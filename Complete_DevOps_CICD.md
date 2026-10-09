# Complete DevOps CI/CD Project — Step-by-Step Lab Guide
## Local VS Code → GitHub → Jenkins → Docker → Docker Hub → Kubernetes (k3s) → Existing AWS EC2

**Audience:** Beginner DevOps students  
**Application:** HTML + CSS + JavaScript served by Nginx  
**Goal:** After initial setup, a developer changes code in normal VS Code, pushes to GitHub, and Jenkins automatically builds a Docker image and deploys it to Kubernetes.

---

# 0. First decision: new EC2 or the same EC2?

## Recommended for your existing training setup

**Reuse the existing EC2 instance that already runs Kubernetes (k3s) and was used in your Kubernetes + Terraform lab. Do not create a new Kubernetes EC2 instance for this project.**

Your Kubernetes Day 2 lab already uses:

| Item | Existing value |
|---|---|
| EC2 | Existing Ubuntu server from the Kubernetes/Terraform lab |
| Kubernetes | k3s |
| Namespace | `devops-demo` |
| Deployment | `webapp` |
| Service | `webapp-service` |
| Browser port | `30080` |
| Browser URL | `http://EC2_PUBLIC_IP:30080` |

Terraform provisioned infrastructure in the earlier lab. For this project, reuse that infrastructure; do not run Terraform again just to publish a new website version.

## Where should Jenkins run?

Use the Jenkins server you already have, if it is working.

- **If Jenkins is already installed on the Kubernetes EC2:** You may use that same EC2 for this learning project if it has sufficient CPU/RAM/disk and Jenkins, Docker build tools and Kubernetes access work.
- **If Jenkins is on a different EC2:** Keep Jenkins there and deploy to the existing Kubernetes EC2. Configure secure network access and Kubernetes credentials from Jenkins to the k3s cluster.
- **Do not launch another EC2 just for the application.** A separate Jenkins server is optional, not required for the application itself.

For production, Jenkins and Kubernetes are normally separated and use dedicated, least-privilege identities. For a beginner lab, reusing the existing environment avoids unnecessary cost and setup.

**Important:** k3s normally uses containerd, not Docker Engine, to run Kubernetes containers. Jenkins can build an image with Docker and push it to Docker Hub; k3s then pulls that image from Docker Hub. Jenkins does not need to copy the image directly into k3s.

---

# 1. Understand the complete flow

```text
YOUR LOCAL COMPUTER
┌──────────────────────────────┐
│ VS Code                      │
│ index.html, style.css, JS    │
│ Dockerfile, Jenkinsfile      │
└──────────────┬───────────────┘
               │ git push
               ▼
┌──────────────────────────────┐
│ GitHub repository            │
└──────────────┬───────────────┘
               │ webhook
               ▼
┌──────────────────────────────┐
│ Jenkins server / agent        │
│ 1. Checkout source            │
│ 2. Validate files             │
│ 3. Build Docker image         │
│ 4. Push image to Docker Hub   │
│ 5. Run kubectl deployment     │
└──────────────┬───────────────┘
               │ image tag + Deployment update
               ▼
EXISTING KUBERNETES EC2
┌──────────────────────────────┐
│ k3s Kubernetes cluster        │
│ Deployment: webapp             │
│   ├── Pod 1: new image         │
│   └── Pod 2: new image         │
│ Service: webapp-service        │
│ NodePort: 30080                │
└──────────────┬───────────────┘
               ▼
       Browser on port 30080
```

## Which step happens where?

| Step | Where to perform it |
|---|---|
| Create/edit application files | Your local computer in VS Code |
| Git commands | VS Code terminal on your local computer |
| Create repository/webhook | GitHub website |
| Create image repository | Docker Hub website |
| Add Jenkins credentials/job | Jenkins website |
| Install/check k3s, inspect Pods and Service | Existing Kubernetes EC2 over SSH |
| Run automated build and deployment | Jenkins agent |
| Verify the live website | Browser on your computer |

Do not confuse the **local VS Code terminal** with the **EC2 SSH terminal**. They are different machines.

---

# 2. Prerequisites checklist

Before changing anything, confirm you have:

- [ ] An existing Ubuntu EC2 instance where k3s is working.
- [ ] The EC2 SSH private key and correct username (usually `ubuntu` for Ubuntu AMIs).
- [ ] The EC2 public IP or a stable DNS name.
- [ ] The existing Jenkins URL and administrator access.
- [ ] Git installed on your local computer.
- [ ] Docker available locally for testing (optional but recommended).
- [ ] A GitHub account.
- [ ] A Docker Hub account.
- [ ] Permission to configure Jenkins credentials and jobs.
- [ ] Secure Kubernetes access from the Jenkins build agent.

If Jenkins is not yet installed or is not working, fix that first. This guide begins after the Jenkins service is accessible.

---

# 3. Step 1 — Check the existing Kubernetes EC2

**Where:** Your local computer, then the SSH session to the existing Kubernetes EC2.

## 3.1 Connect to EC2

Open PowerShell or a terminal on your local computer. Use the key and public IP for your existing Kubernetes/Terraform server:

```bash
ssh -i "YOUR_KEY.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace both placeholders with your real values. If Windows PowerShell cannot find the key, use its full path. Do not upload or share the private key.

## 3.2 Check the operating system and user

Run **inside the EC2 SSH session**:

```bash
whoami
hostname
cat /etc/os-release
```

For Ubuntu, the username should usually be `ubuntu`.

## 3.3 Check Kubernetes

Run **inside the EC2 SSH session**:

```bash
kubectl get nodes
kubectl get all -n devops-demo
kubectl get deployment webapp -n devops-demo
kubectl get service webapp-service -n devops-demo
```

The node should show `Ready`. The Deployment should have its desired Pods available.

## 3.4 Test the current website

On your local computer, open:

```text
http://YOUR_EC2_PUBLIC_IP:30080
```

If the current application is not reachable, troubleshoot it before proceeding.

## 3.5 Back up current resources

Run **inside the EC2 SSH session**:

```bash
mkdir -p ~/k8s-backup-before-cicd

kubectl get deployment webapp -n devops-demo -o yaml \
  > ~/k8s-backup-before-cicd/webapp-deployment.yaml

kubectl get service webapp-service -n devops-demo -o yaml \
  > ~/k8s-backup-before-cicd/webapp-service.yaml
```

These are backup references for recovery. They may contain Kubernetes-generated fields.

---

# 4. Step 2 — Check the EC2 Security Group

**Where:** AWS Console in your browser.

Open **EC2 → Instances → select the existing Kubernetes EC2 → Security tab → Security groups**.

Verify the inbound rules required by your lab:

- **SSH, TCP 22:** restrict to your own public IP where possible.
- **Application, TCP 30080:** allow only intended viewer IPs. For a temporary classroom demonstration, a broader rule may be used if appropriate, but restrict it afterward.
- **Jenkins web UI:** only if Jenkins runs on this instance; restrict access to trusted IPs or a private network and use HTTPS/reverse proxy where configured.

Do not add an inbound rule for the Kubernetes API just to make Jenkins work. If Jenkins is on another server, prefer private networking and secure, limited access. Do not expose the Docker daemon publicly.

---

# 5. Step 3 — Create the application on your local computer

**Where:** Local computer → VS Code, not the EC2 SSH terminal.

Create a folder called `devops-cicd-webapp` and open it in VS Code. Create:

```text
devops-cicd-webapp/
├── index.html
├── style.css
├── script.js
├── Dockerfile
├── .dockerignore
├── Jenkinsfile
├── deployment.yaml
└── README.md
```

Do not add passwords, Docker Hub tokens, GitHub tokens, SSH keys or Kubernetes kubeconfig files.

## 5.1 Create `index.html`

Paste into the local VS Code file `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>DevOps CI/CD Demo</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <main class="card">
    <p class="eyebrow">AUTOMATED DELIVERY</p>
    <h1>My DevOps Application</h1>
    <h2 id="version">Version 1</h2>
    <p>This website is delivered through GitHub, Jenkins, Docker and Kubernetes.</p>
    <button id="testButton">Test Application</button>
    <p id="message" role="status"></p>
    <footer>Running on Kubernetes on AWS EC2</footer>
  </main>
  <script src="script.js"></script>
</body>
</html>
```

## 5.2 Create `style.css`

Paste into `style.css`:

```css
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  min-height: 100vh;
  display: grid;
  place-items: center;
  padding: 24px;
  font-family: Arial, sans-serif;
  background: #eef3f9;
  color: #172033;
}

.card {
  width: min(720px, 100%);
  padding: 44px 32px;
  text-align: center;
  background: #ffffff;
  border-radius: 18px;
  box-shadow: 0 12px 40px rgba(25, 45, 80, 0.12);
}

.eyebrow {
  color: #2563eb;
  font-weight: bold;
  letter-spacing: 2px;
}

h1 {
  font-size: clamp(30px, 5vw, 46px);
}

h2 {
  color: #2563eb;
}

p {
  line-height: 1.7;
}

button {
  margin: 18px 0;
  padding: 13px 22px;
  border: 0;
  border-radius: 8px;
  background: #2563eb;
  color: #ffffff;
  font-size: 16px;
  cursor: pointer;
}

button:hover {
  background: #1d4ed8;
}

footer {
  margin-top: 28px;
  font-size: 13px;
  color: #64748b;
}
```

## 5.3 Create `script.js`

Paste into `script.js`:

```javascript
const button = document.getElementById("testButton");
const message = document.getElementById("message");

button.addEventListener("click", () => {
  message.textContent = "Success! The application is responding.";
});
```

## 5.4 Create `Dockerfile`

Paste into `Dockerfile`:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
COPY style.css /usr/share/nginx/html/style.css
COPY script.js /usr/share/nginx/html/script.js

EXPOSE 80
```

**Explanation:** The Docker image contains Nginx and all three website files. Kubernetes will run this image. We will no longer mount the old website ConfigMap over Nginx's web directory.

## 5.5 Create `.dockerignore`

```text
.git
.gitignore
README.md
```

## 5.6 Create `README.md`

```markdown
# DevOps CI/CD Web Application

A static website deployed through a CI/CD pipeline.

## Technologies
- HTML, CSS, JavaScript
- Git and GitHub
- Jenkins
- Docker
- Docker Hub
- Kubernetes (k3s)
- AWS EC2

## Flow
VS Code -> GitHub -> Jenkins -> Docker Hub -> Kubernetes -> EC2

## Local test
docker build -t devops-cicd-webapp:local .
docker run --rm -p 8081:80 devops-cicd-webapp:local

Open http://localhost:8081
```

Save all files.

---

# 6. Step 4 — Test Git and push the source code

**Where:** VS Code terminal on your local computer.

Open **Terminal → New Terminal** in VS Code. Confirm the current directory is the project folder:

```bash
pwd
git --version
```

Confirm the folder contains `index.html`, `style.css`, `script.js` and `Dockerfile`. If Git is not installed, install Git for your operating system and reopen VS Code.

## 6.1 Create the GitHub repository

**Where:** GitHub website.

1. Sign in to GitHub.
2. Click **New repository**.
3. Name it `devops-cicd-webapp`.
4. Choose public or private.
5. If your local folder already contains project files, do not initialize the remote with a README, license or `.gitignore`.
6. Create the repository and copy its HTTPS URL.

## 6.2 Initialize and push

**Where:** VS Code terminal in the project folder.

```bash
git init
git branch -M main
git add .
git status
git commit -m "Initial website and Docker setup"
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/devops-cicd-webapp.git
git push -u origin main
```

Replace `YOUR_GITHUB_USERNAME`.

If `git remote add origin` reports that `origin` already exists, inspect it:

```bash
git remote -v
```

If it points to the correct repository, do not add it again. If it points to the wrong repository, correct it:

```bash
git remote set-url origin https://github.com/YOUR_GITHUB_USERNAME/devops-cicd-webapp.git
```

If Git asks for authentication, use GitHub's supported authentication method. Do not put a personal access token in the Git remote URL or source files.

## 6.3 Verify

**Where:** GitHub website.

Refresh the repository page. You should see the project files. Continue only after the push succeeds.

---

# 7. Step 5 — Create the Docker Hub image repository

**Where:** Docker Hub website.

1. Sign in to Docker Hub.
2. Create a repository named `devops-cicd-webapp`.
3. Record your Docker Hub username.
4. Choose public or private.

A public repository is simpler for a first lab because k3s can pull the image without a pull Secret. A private repository is possible, but you must configure `imagePullSecrets` in the `devops-demo` namespace.

Never commit Docker Hub credentials to GitHub.

---

# 8. Step 6 — Confirm Jenkins build-agent tools

**Where:** The machine/agent that actually runs Jenkins pipeline shell commands. It might be the Jenkins EC2 or a separate agent; do not assume it is the Kubernetes EC2.

On that agent, verify:

```bash
git --version
docker --version
docker info
kubectl version --client
kubectl get nodes
kubectl get deployment webapp -n devops-demo
```

The first four commands check tools. The last two check access to the intended cluster.

If `docker info` fails, Docker daemon access is not ready. If `kubectl get nodes` fails, cluster authentication or network access is not ready. Fix those issues before creating an automatic deployment.

**Security:** Access to the Docker daemon can grant root-equivalent control of the host. Use a controlled Jenkins agent. Do not expose an unauthenticated Docker TCP socket.

## 8.1 Kubernetes credentials for Jenkins

### If Jenkins runs on the same EC2 as k3s

k3s commonly stores its administrator kubeconfig at:

```text
/etc/rancher/k3s/k3s.yaml
```

Do not make it world-readable and do not commit it to GitHub. Configure a secure kubeconfig for the Jenkins job user or use a dedicated, least-privilege Kubernetes identity. Test commands as the same OS user that executes the job.

### If Jenkins runs on a different EC2

Configure a secure kubeconfig or equivalent Kubernetes authentication for the Jenkins agent. Ensure the API endpoint is reachable over a controlled network path. Prefer private networking and least-privilege permissions. Do not open the Kubernetes API to the entire internet.

The pipeline should not proceed until `kubectl get nodes` and `kubectl get deployment webapp -n devops-demo` work from the pipeline agent.

---

# 9. Step 7 — Add Docker Hub credentials in Jenkins

**Where:** Jenkins website.

1. Open Jenkins.
2. Go to **Manage Jenkins → Credentials**.
3. Select the appropriate credential store/domain, usually **System → Global credentials** for a simple lab.
4. Click **Add Credentials**.
5. Choose **Kind: Username with password**.
6. Username: your Docker Hub username.
7. Password: a Docker Hub access token with the minimum permissions required to push.
8. ID: enter exactly `dockerhub-creds`.
9. Add a description such as `Docker Hub push token`.
10. Save.

The pipeline refers to this credential by ID. It does not store the token in GitHub.

---

# 10. Step 8 — Build and push the first image

Do this once before the first Kubernetes image Deployment. It confirms the Dockerfile works and makes an image available to k3s.

## 10.1 Test locally

**Where:** VS Code terminal on your local computer, inside the project folder. Docker must be installed and running locally.

Replace `YOUR_DOCKERHUB_USERNAME`:

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/devops-cicd-webapp:initial .
docker run --rm -p 8081:80 YOUR_DOCKERHUB_USERNAME/devops-cicd-webapp:initial
```

Open this URL on your local computer:

```text
http://localhost:8081
```

Test the button. Stop the foreground container with `Ctrl+C`.

## 10.2 Log in and push

**Where:** VS Code terminal on your local computer.

```bash
docker login
docker push YOUR_DOCKERHUB_USERNAME/devops-cicd-webapp:initial
```

Use a secure authentication prompt. Do not paste a token into source code or save it in a script.

## 10.3 Verify the image

**Where:** Docker Hub website.

Open your `devops-cicd-webapp` repository and verify that the `initial` tag exists.

If local Docker is unavailable, the initial image can be built and pushed by a properly configured Jenkins agent, but complete the first image push before switching Kubernetes to the new image.

---

# 11. Step 9 — Change Kubernetes from the old ConfigMap app to the Docker image

**Where:** Local VS Code for the manifest; existing Kubernetes EC2 SSH session for `kubectl` commands.

The Day 2 project mounted `web-content` over `/usr/share/nginx/html`. If that mount remains, it can hide the HTML/CSS/JS packaged in the Docker image. The new Deployment must use the image and must not mount that old website ConfigMap at the Nginx web root.

## 11.1 Create `deployment.yaml` locally

In the root of your local VS Code project, create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: devops-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: nginx
          image: YOUR_DOCKERHUB_USERNAME/devops-cicd-webapp:initial
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

Replace the Docker Hub username. Check indentation carefully: YAML indentation is significant.

This manifest keeps the existing Deployment name, namespace, selector, two replicas and container name `nginx`. It deliberately has no ConfigMap volume.

## 11.2 Copy or apply the manifest

**Recommended:** Commit this manifest to GitHub, then use it as the source of truth. To apply it immediately for the one-time migration, copy the file to the Kubernetes EC2 using an approved transfer method, or use the existing repository checkout on that server.

If using `scp` from your local terminal, run:

```bash
scp -i "YOUR_KEY.pem" deployment.yaml ubuntu@YOUR_EC2_PUBLIC_IP:/home/ubuntu/deployment.yaml
```

Replace the key path and IP. Then SSH into the existing Kubernetes EC2 and run:

```bash
kubectl apply -f /home/ubuntu/deployment.yaml
kubectl rollout status deployment/webapp -n devops-demo
kubectl get deployment webapp -n devops-demo
kubectl get pods -n devops-demo
```

**Only apply this after the `initial` image exists in Docker Hub.**

## 11.3 Keep the existing Service

On the Kubernetes EC2, check:

```bash
kubectl get service webapp-service -n devops-demo -o yaml
```

It should select Pods with `app: webapp` and expose the application on NodePort `30080`. Do not create a second Service if the existing one is correct.

## 11.4 Test the migrated application

Open on your local computer:

```text
http://YOUR_EC2_PUBLIC_IP:30080
```

You should see Version 1. If it does not work, check Pod events and image-pull status before continuing.

---

# 12. Step 10 — Create the Jenkinsfile

**Where:** Local VS Code, in the root of `devops-cicd-webapp`.

Create a file named exactly `Jenkinsfile` (no `.txt` extension). Replace `YOUR_DOCKERHUB_USERNAME`.

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_REPO = 'YOUR_DOCKERHUB_USERNAME/devops-cicd-webapp'
        DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
        K8S_NAMESPACE = 'devops-demo'
        K8S_DEPLOYMENT = 'webapp'
        K8S_CONTAINER = 'nginx'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                    env.FULL_IMAGE = "${env.IMAGE_REPO}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Validate Source') {
            steps {
                sh '''
                    set -eu
                    test -f index.html
                    test -f style.css
                    test -f script.js
                    test -f Dockerfile
                    echo "Required files are present."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build --pull -t "$FULL_IMAGE" .
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIALS}",
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" --password-stdin
                        docker push "$FULL_IMAGE"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu
                    kubectl set image \
                      deployment/"$K8S_DEPLOYMENT" \
                      "$K8S_CONTAINER"="$FULL_IMAGE" \
                      -n "$K8S_NAMESPACE"

                    kubectl rollout status \
                      deployment/"$K8S_DEPLOYMENT" \
                      -n "$K8S_NAMESPACE" \
                      --timeout=180s

                    kubectl get deployment "$K8S_DEPLOYMENT" \
                      -n "$K8S_NAMESPACE" -o wide
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Inspect the failed stage and console output.'
        }
    }
}
```

## How this pipeline works

- `Checkout`: downloads the code revision selected by the Jenkins job.
- `Validate Source`: checks that the expected files exist.
- `Build Docker Image`: builds an image from the Dockerfile.
- `Push Image to Docker Hub`: authenticates using the Jenkins credential and pushes a versioned image.
- `Deploy to Kubernetes`: changes the image used by the existing Deployment and waits for rollout completion.

The tag is based on the Git commit ID, so each commit creates a traceable image version rather than relying only on `latest`.

**Assumptions to verify:** the pipeline agent is Linux-based, supports `sh`, has Docker daemon access, has `kubectl` installed and authenticated, and the Kubernetes container is named `nginx`. The pipeline will fail until all of these are true.

---

# 13. Step 11 — Push the Jenkinsfile and Deployment manifest

**Where:** VS Code terminal on your local computer.

```bash
git add Jenkinsfile deployment.yaml
git commit -m "Add Jenkins CI/CD pipeline and Kubernetes deployment"
git push origin main
```

Open GitHub and confirm both files are present.

---

# 14. Step 12 — Create the Jenkins Pipeline job

**Where:** Jenkins website.

1. Click **New Item**.
2. Enter `devops-cicd-webapp-pipeline`.
3. Select **Pipeline**.
4. Click **OK**.
5. In the Pipeline section, choose **Pipeline script from SCM**.
6. SCM: choose **Git**.
7. Repository URL: `https://github.com/YOUR_GITHUB_USERNAME/devops-cicd-webapp.git`.
8. If the repository is private, select the Jenkins GitHub read credential.
9. Branch Specifier: `*/main`.
10. Script Path: `Jenkinsfile`.
11. Save.

If Jenkins does not have the Git plugin or Pipeline support installed, install the appropriate official Jenkins plugins through **Manage Jenkins → Plugins** and restart Jenkins only if prompted.

## 14.1 Run the first pipeline manually

**Where:** Jenkins website.

1. Open `devops-cicd-webapp-pipeline`.
2. Click **Build Now**.
3. Open the new build.
4. Click **Console Output**.
5. Watch each stage.

The first run must finish successfully before configuring automatic deployment. If it fails, identify the exact stage and fix that stage first.

Expected stages:

```text
Checkout
Validate Source
Build Docker Image
Push Image to Docker Hub
Deploy to Kubernetes
Finished: SUCCESS
```

A successful pipeline should show that the image was pushed and the Kubernetes rollout completed.

---

# 15. Step 13 — Configure automatic GitHub webhook builds

A webhook tells Jenkins that code was pushed. Without a working webhook, students may need to click **Build Now** manually.

## 15.1 Configure the Jenkins job

**Where:** Jenkins website → `devops-cicd-webapp-pipeline` → **Configure**.

Under **Build Triggers**, enable **GitHub hook trigger for GITScm polling** if this option is available in your Jenkins installation. Save the job.

## 15.2 Add the webhook in GitHub

**Where:** GitHub website → repository → **Settings → Webhooks → Add webhook**.

Use the externally reachable Jenkins webhook endpoint, commonly:

```text
https://YOUR_JENKINS_HOST/github-webhook/
```

Set:
- Content type: `application/json`
- Events: push events
- Active: enabled

Use the actual reachable Jenkins URL for your environment. The URL above is an example, not a real link.

Use HTTPS and restrict Jenkins access appropriately. Do not expose an unsecured Jenkins server to the internet. If Jenkins is only reachable on a private network, use an approved secure connectivity method rather than opening it broadly.

## 15.3 Test the webhook

**Where:** GitHub website.

Open the webhook's **Recent Deliveries** and inspect the response after the next push. If it fails, check the URL, connectivity, Jenkins trigger configuration and logs. A manual **Build Now** can still test the pipeline while troubleshooting.

---

# 16. Step 14 — Prove the end-to-end workflow

This is the main classroom demonstration. From this point, students should make application changes only in local VS Code.

## 16.1 Change the application

**Where:** Local VS Code.

Open `index.html`. Change:

```html
<h2 id="version">Version 1</h2>
```

to:

```html
<h2 id="version">Version 2 - CI/CD Successful</h2>
```

Save the file.

## 16.2 Push the change

**Where:** VS Code terminal on your local computer.

```bash
git add index.html
git commit -m "Update website to version 2"
git push origin main
```

## 16.3 Watch the pipeline

**Where:** Jenkins website.

The GitHub webhook should trigger a new build. Open the build and Console Output.

Observe:
1. Jenkins checks out the latest commit.
2. Jenkins validates files.
3. Docker builds a new image.
4. Jenkins pushes the commit-tagged image to Docker Hub.
5. Jenkins updates the Kubernetes Deployment.
6. Kubernetes replaces old Pods and completes the rollout.

## 16.4 Verify the live application

**Where:** Browser on your local computer.

Open or refresh:

```text
http://YOUR_EC2_PUBLIC_IP:30080
```

The website should now show `Version 2 - CI/CD Successful`.

**Success condition:** You changed code in VS Code, pushed it to GitHub, Jenkins ran automatically, the new image was deployed to Kubernetes on the existing EC2, and the browser showed the new version.

---

# 17. Step 15 — Troubleshooting

## A. GitHub push does not start Jenkins

Check:
- The webhook URL is correct and reachable.
- GitHub webhook delivery reports success.
- The Jenkins job has the webhook trigger enabled.
- The job points to `main`.
- Jenkins is not blocked by firewall or reverse-proxy configuration.

Use **Build Now** to test the pipeline separately from the webhook.

## B. `docker: command not found`

Docker CLI is missing on the agent that runs the pipeline. Install/configure it on the intended agent or use a dedicated build agent with Docker support.

## C. Docker permission denied

The Jenkins OS user cannot access the Docker daemon. Fix the agent's permissions securely. Do not expose an unauthenticated Docker TCP socket.

## D. Docker Hub authentication/push fails

Check:
- Repository name and username.
- Jenkins credential ID is exactly `dockerhub-creds`.
- The token has permission to push.
- The Docker Hub repository exists.
- The credential is selected correctly in Jenkins.

## E. `kubectl` cannot connect

Run on the pipeline agent as the same user that runs the job:

```bash
kubectl config current-context
kubectl get nodes
kubectl get deployment webapp -n devops-demo
```

Check kubeconfig, API endpoint, credentials and network connectivity. Do not make administrator kubeconfig files world-readable or open the Kubernetes API to all IPs.

## F. `ImagePullBackOff`

Run **on the Kubernetes EC2**:

```bash
kubectl get pods -n devops-demo
kubectl describe pods -n devops-demo
```

Check the image repository and tag, repository visibility and pull Secret if the Docker Hub repository is private.

## G. Deployment does not finish

Run on the Kubernetes EC2:

```bash
kubectl get pods -n devops-demo
kubectl describe deployment webapp -n devops-demo
kubectl get events -n devops-demo --sort-by=.metadata.creationTimestamp
```

Read Pod events and look for image-pull errors, scheduling problems or startup failures.

## H. Website still shows the old version

Run on the Kubernetes EC2:

```bash
kubectl get deployment webapp -n devops-demo \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

kubectl rollout status deployment/webapp -n devops-demo
kubectl get endpoints webapp-service -n devops-demo
```

Check that the Deployment references the expected commit-tagged image, Jenkins pushed the new image, the rollout completed, the Service selector matches the Pod labels, and the browser was refreshed.

## I. CSS or JavaScript does not load

The HTML references `style.css` and `script.js`; the Dockerfile must copy those files to `/usr/share/nginx/html/`. Check file spelling and letter case. Linux filenames are case-sensitive.

---

# 18. Step 16 — Practise rollback

If a release is faulty and the previous Deployment revision is available, run on the Kubernetes EC2:

```bash
kubectl rollout history deployment/webapp -n devops-demo
kubectl rollout undo deployment/webapp -n devops-demo
kubectl rollout status deployment/webapp -n devops-demo
```

Verify the live page afterward.

Rollback requires the earlier Deployment revision and its image to remain available. Do not delete older image tags until they are no longer needed.

For a classroom exercise:
1. Deploy a working Version 2.
2. Make a faulty Version 3 change.
3. Push it and inspect the pipeline result.
4. Diagnose the failure.
5. Roll back to the previous working Deployment revision.
6. Confirm the website is working again.

A faulty HTML/CSS change might still pass basic file validation. This is useful for teaching the difference between a successful pipeline and a correct application. Later, add automated tests to catch more application errors before deployment.

---

# 19. Student completion checklist

- [ ] Reused the existing Kubernetes/Terraform EC2 rather than creating another Kubernetes server.
- [ ] Verified k3s node, Deployment, Pods and Service.
- [ ] Backed up the existing Deployment and Service YAML.
- [ ] Created HTML, CSS, JavaScript and Dockerfile in local VS Code.
- [ ] Pushed the project to GitHub.
- [ ] Created a Docker Hub repository.
- [ ] Confirmed Jenkins agent has Git, Docker and kubectl access.
- [ ] Stored Docker Hub credentials in Jenkins.
- [ ] Built and pushed the initial image.
- [ ] Removed the old ConfigMap mount from the application Deployment.
- [ ] Confirmed the existing Service still uses NodePort `30080`.
- [ ] Added `Jenkinsfile` and `deployment.yaml` to GitHub.
- [ ] Created a Jenkins Pipeline job.
- [ ] Ran the first pipeline manually and reviewed Console Output.
- [ ] Configured the GitHub webhook.
- [ ] Changed Version 1 to Version 2 in VS Code.
- [ ] Pushed the code and observed an automatic Jenkins build.
- [ ] Verified the new image tag and successful Kubernetes rollout.
- [ ] Opened the website using the existing EC2 public IP.
- [ ] Completed troubleshooting and rollback practice.

---

# 20. Interview questions

1. What is the difference between CI and CD?
2. Why use a Docker image instead of copying files directly into a running Pod?
3. What does a GitHub webhook do?
4. Why use a Git commit ID as an image tag?
5. What is Docker Hub used for?
6. What is the difference between a Pod, Deployment and Service?
7. What does `kubectl rollout status` verify?
8. What causes `ImagePullBackOff`?
9. Why should credentials be stored in Jenkins instead of GitHub?
10. Why can a Pod be `Running` while the website is still incorrect?
11. How would you troubleshoot a failed pipeline?
12. Why can we reuse the EC2 server rather than create a new one for every application update?

---

# Final reminder

**Reuse the existing EC2 instance that already hosts k3s and was used for Kubernetes/Terraform.** Keep using the Jenkins server you already have, provided it can build images and securely access the target Kubernetes cluster. Do not run Terraform again just to publish an application code change.

The regular developer workflow after setup is:

```text
Edit in VS Code
    ↓
git add / git commit / git push
    ↓
GitHub webhook
    ↓
Jenkins pipeline
    ↓
Docker build + push to Docker Hub
    ↓
Kubernetes Deployment update on existing EC2
    ↓
Browser verifies the new application
```
