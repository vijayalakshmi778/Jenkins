# Jenkins Beginner Practical Project

## GitHub → Jenkins → Docker → AWS EC2 → Web Application

This is a beginner-friendly first Jenkins project.

You will connect the topics already learned:

```text
Linux → AWS EC2 → Git/GitHub → Docker → Jenkins
```

## 1. Project Goal

You will create a simple HTML website, push it to GitHub, and use Jenkins to deploy it inside a Docker container on an AWS EC2 Ubuntu server.

Final flow:

```text
Developer
    |
    | git push
    v
 GitHub
    |
    | Jenkins gets latest code
    v
 Jenkins
    |
    | docker build
    v
Docker Image
    |
    | docker run
    v
Docker Container
    |
    v
Nginx Web Server
    |
    v
AWS EC2
    |
    v
Web Browser
```

Final website:

```text
http://EC2-PUBLIC-IP:8081
```

---

# 2. What Is Jenkins?

Jenkins is an automation server.

Without Jenkins, deployment can require manually doing:

```text
Get code
   ↓
Build application
   ↓
Build Docker image
   ↓
Stop old container
   ↓
Start new container
   ↓
Check application
```

Jenkins can automate these steps.

In this project:

```text
GitHub
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Container
   ↓
AWS EC2
   ↓
Website
```

---

# 3. Requirements

You need:

- AWS account
- GitHub account
- Ubuntu EC2 instance
- SSH key
- Git installed on your local computer
- Basic Linux knowledge
- Basic Git/GitHub knowledge
- Basic Docker knowledge

---

# 4. Create Ubuntu EC2 Instance

Create an Ubuntu EC2 instance in AWS.

For this beginner lab, use a small instance suitable for your classroom environment.

After creating the instance, note the:

```text
Public IPv4 address
```

Example:

```text
13.234.XX.XX
```

---

# 5. Configure EC2 Security Group

Go to:

```text
AWS Console
→ EC2
→ Instances
→ Your Instance
→ Security
→ Security Groups
→ Inbound Rules
```

Add:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | My IP | SSH |
| TCP | 8080 | My IP | Jenkins |
| TCP | 8081 | 0.0.0.0/0 | Website |

Jenkins uses port `8080`.

The website will use port `8081`.

This avoids a port conflict.

---

# 6. Connect to EC2

Open PowerShell.

Go to the folder containing your `.pem` file.

Run:

```bash
ssh -i "your-key.pem" ubuntu@EC2-PUBLIC-IP
```

Example:

```bash
ssh -i "mykey.pem" ubuntu@13.234.XX.XX
```

After successful login:

```text
ubuntu@ip-172-31-xx-xx:~$
```

You are now inside Ubuntu.

---

# 7. Update Ubuntu

Run:

```bash
sudo apt update
```

Then:

```bash
sudo apt upgrade -y
```

---

# 8. Install Java

Jenkins requires Java.

Run:

```bash
sudo apt install fontconfig openjdk-21-jre -y
```

Check:

```bash
java -version
```

You should see Java version information.

---

# 9. Install Jenkins

Add the Jenkins repository key:

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```

Add the repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Update packages:

```bash
sudo apt update
```

Install Jenkins:

```bash
sudo apt install jenkins -y
```

---

# 10. Start Jenkins

Run:

```bash
sudo systemctl start jenkins
```

Enable Jenkins after reboot:

```bash
sudo systemctl enable jenkins
```

Check status:

```bash
sudo systemctl status jenkins
```

Look for:

```text
Active: active (running)
```

Press:

```text
q
```

to exit.

---

# 11. Open Jenkins

Check port:

```bash
sudo ss -lntp | grep 8080
```

Open your browser:

```text
http://EC2-PUBLIC-IP:8080
```

Example:

```text
http://13.234.XX.XX:8080
```

---

# 12. Get Jenkins Initial Password

In the EC2 terminal:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the password.

Paste it into the Jenkins page.

Click:

```text
Continue
```

---

# 13. Install Suggested Plugins

Select:

```text
Install suggested plugins
```

Wait for the installation to finish.

---

# 14. Create Jenkins Administrator

Enter your details.

Example:

```text
Username: jenkinsadmin
Password: YOUR-PASSWORD
Full Name: Your Name
Email: your-email@example.com
```

Click:

```text
Save and Continue
```

Then:

```text
Start using Jenkins
```

You should see the Jenkins dashboard.

---

# 15. Install Git on EC2

Check Git:

```bash
git --version
```

If Git is missing:

```bash
sudo apt install git -y
```

Check again:

```bash
git --version
```

---

# 16. Install Docker

Check Docker:

```bash
docker --version
```

If Docker is missing:

```bash
sudo apt update
```

Then:

```bash
sudo apt install docker.io -y
```

Start Docker:

```bash
sudo systemctl start docker
```

Enable Docker:

```bash
sudo systemctl enable docker
```

Check:

```bash
sudo systemctl status docker
```

Press `q` to exit.

---

# 17. Test Docker

Run:

```bash
sudo docker run hello-world
```

If the Hello World message appears, Docker is working.

---

# 18. Allow Jenkins to Use Docker

Jenkins runs as the Linux user `jenkins`.

Add Jenkins to the Docker group:

```bash
sudo usermod -aG docker jenkins
```

Restart Jenkins:

```bash
sudo systemctl restart jenkins
```

Test:

```bash
sudo -u jenkins docker ps
```

If there is no permission error, Jenkins can use Docker.

---

# 19. Create the Website Project

On your local computer create:

```text
jenkins-docker-project/
│
├── index.html
└── Dockerfile
```

---

# 20. Create `index.html`

Put this inside `index.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Jenkins Docker Project</title>
</head>
<body>

    <h1>Jenkins Deployment Successful!</h1>

    <p>This website was deployed using Jenkins and Docker.</p>

    <p>GitHub → Jenkins → Docker → AWS EC2</p>

</body>
</html>
```

Save the file.

---

# 21. Create `Dockerfile`

Create a file named exactly:

```text
Dockerfile
```

Do not create:

```text
Dockerfile.txt
```

Add:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

---

# 22. Understand the Dockerfile

```dockerfile
FROM nginx:alpine
```

Uses Nginx as the web server.

```dockerfile
COPY index.html /usr/share/nginx/html/index.html
```

Copies the website into Nginx.

```dockerfile
EXPOSE 80
```

Documents the container's web port.

---

# 23. Test the Docker Project Locally

Open PowerShell inside the project folder.

Check the files:

```powershell
dir
```

You should see:

```text
Dockerfile
index.html
```

---

# 24. Build the Docker Image

Run:

```bash
docker build -t jenkins-web-app .
```

Check:

```bash
docker images
```

You should see:

```text
jenkins-web-app
```

---

# 25. Run the Website Locally

Run:

```bash
docker run -d --name jenkins-web-container -p 8081:80 jenkins-web-app
```

Port mapping:

```text
Host 8081
   ↓
Container 80
```

Open:

```text
http://localhost:8081
```

You should see:

```text
Jenkins Deployment Successful!
```

---

# 26. Check the Container

Run:

```bash
docker ps
```

You should see:

```text
jenkins-web-container
```

---

# 27. Remove the Local Test Container

Stop it:

```bash
docker stop jenkins-web-container
```

Remove it:

```bash
docker rm jenkins-web-container
```

This was only a local test.

---

# 28. Create GitHub Repository

Go to GitHub.

Create a repository:

```text
jenkins-docker-project
```

For this beginner project, you can make it public.

Create the repository.

---

# 29. Initialize Git

Open terminal inside the project folder.

Run:

```bash
git init
```

Check:

```bash
git status
```

---

# 30. Add Project Files

Run:

```bash
git add .
```

Check:

```bash
git status
```

You should see:

```text
Dockerfile
index.html
```

---

# 31. Create First Commit

Run:

```bash
git commit -m "Initial Jenkins Docker project"
```

---

# 32. Rename Branch to `main`

Run:

```bash
git branch -M main
```

Check:

```bash
git branch
```

Expected:

```text
* main
```

---

# 33. Connect GitHub Repository

Copy your GitHub repository URL.

Example:

```text
https://github.com/YOUR-USERNAME/jenkins-docker-project.git
```

Run:

```bash
git remote add origin https://github.com/YOUR-USERNAME/jenkins-docker-project.git
```

Check:

```bash
git remote -v
```

---

# 34. Push to GitHub

Run:

```bash
git push -u origin main
```

Open GitHub.

You should see:

```text
Dockerfile
index.html
```

Do not continue until these files are visible in GitHub.

---

# 35. Create Jenkins Pipeline

Open:

```text
http://EC2-PUBLIC-IP:8080
```

From the Jenkins dashboard click:

```text
New Item
```

Enter:

```text
jenkins-docker-project
```

Select:

```text
Pipeline
```

Click:

```text
OK
```

---

# 36. Configure Pipeline

Scroll down to:

```text
Pipeline
```

Find:

```text
Definition
```

Select:

```text
Pipeline script
```

---

# 37. Add the Pipeline Script

Paste:

```groovy
pipeline {

    agent any

    stages {

        stage('Clone GitHub Repository') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/YOUR-USERNAME/jenkins-docker-project.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t jenkins-web-app .'
            }
        }

        stage('Stop Old Container') {
            steps {
                sh 'docker stop jenkins-web-container || true'
                sh 'docker rm jenkins-web-container || true'
            }
        }

        stage('Run New Container') {
            steps {
                sh 'docker run -d --name jenkins-web-container -p 8081:80 jenkins-web-app'
            }
        }

    }
}
```

---

# 38. Change the GitHub URL

Find:

```groovy
https://github.com/YOUR-USERNAME/jenkins-docker-project.git
```

Replace `YOUR-USERNAME`.

Example:

```groovy
git branch: 'main',
    url: 'https://github.com/student123/jenkins-docker-project.git'
```

Do not change the rest of the pipeline.

Click:

```text
Save
```

---

# 39. Run Jenkins Build #1

Open the Jenkins project.

Click:

```text
Build Now
```

Jenkins will create:

```text
Build #1
```

---

# 40. Open Console Output

Click:

```text
Build #1
```

Then:

```text
Console Output
```

Watch the build.

The pipeline should run:

```text
Clone GitHub Repository
        ↓
Build Docker Image
        ↓
Stop Old Container
        ↓
Run New Container
```

---

# 41. Check Build Result

At the bottom of Console Output, look for:

```text
Finished: SUCCESS
```

If you see this, the pipeline completed successfully.

---

# 42. Check Docker on EC2

SSH into EC2.

Run:

```bash
docker images
```

You should see:

```text
jenkins-web-app
```

Now:

```bash
docker ps
```

You should see:

```text
jenkins-web-container
```

---

# 43. Open the Deployed Website

Open:

```text
http://EC2-PUBLIC-IP:8081
```

Example:

```text
http://13.234.XX.XX:8081
```

You should see:

```text
Jenkins Deployment Successful!
```

Your first Jenkins deployment is complete.

---

# 44. Test Automatic Deployment Manually

Now change the website.

Open:

```text
index.html
```

Change:

```html
<h1>Jenkins Deployment Successful!</h1>
```

to:

```html
<h1>Jenkins Automatic Deployment Successful!</h1>
```

Save the file.

---

# 45. Check Git Status

Run:

```bash
git status
```

You should see:

```text
modified: index.html
```

---

# 46. Commit the Change

Run:

```bash
git add .
```

Then:

```bash
git commit -m "Update website message"
```

---

# 47. Push the Change

Run:

```bash
git push
```

Open GitHub and verify that the updated `index.html` is present.

---

# 48. Run Jenkins Build #2

Go to:

```text
Jenkins
→ jenkins-docker-project
```

Click:

```text
Build Now
```

Jenkins will:

```text
Get latest GitHub code
        ↓
Build new Docker image
        ↓
Stop old container
        ↓
Remove old container
        ↓
Run new container
```

---

# 49. Check Build #2

Open:

```text
Build #2
```

Then:

```text
Console Output
```

Check for:

```text
Finished: SUCCESS
```

---

# 50. Check the Updated Website

Open:

```text
http://EC2-PUBLIC-IP:8081
```

Refresh the browser.

You should now see:

```text
Jenkins Automatic Deployment Successful!
```

This proves that Jenkins deployed the updated code.

---

# 51. Understand the Complete Workflow

```text
Developer
    |
    | git push
    v
GitHub
    |
    | Jenkins gets code
    v
Jenkins
    |
    | docker build
    v
Docker Image
    |
    | docker run
    v
Docker Container
    |
    v
Nginx
    |
    v
AWS EC2
    |
    | :8081
    v
Web Browser
```

---

# 52. What Each Jenkins Stage Does

## Stage 1 — Clone GitHub Repository

```groovy
stage('Clone GitHub Repository') {
    steps {
        git branch: 'main',
            url: 'https://github.com/YOUR-USERNAME/jenkins-docker-project.git'
    }
}
```

Jenkins gets the latest code from GitHub.

---

## Stage 2 — Build Docker Image

```groovy
stage('Build Docker Image') {
    steps {
        sh 'docker build -t jenkins-web-app .'
    }
}
```

Jenkins tells Docker to build the image.

---

## Stage 3 — Stop Old Container

```groovy
stage('Stop Old Container') {
    steps {
        sh 'docker stop jenkins-web-container || true'
        sh 'docker rm jenkins-web-container || true'
    }
}
```

Jenkins removes the previous running version.

`|| true` prevents the pipeline from failing if the container does not exist during the first deployment.

---

## Stage 4 — Run New Container

```groovy
stage('Run New Container') {
    steps {
        sh 'docker run -d --name jenkins-web-container -p 8081:80 jenkins-web-app'
    }
}
```

Jenkins starts the latest Docker image.

---

# 53. Important Jenkins Terms

### Jenkins

Automation server.

### Job

A Jenkins task/project.

### Pipeline

A sequence of automated stages.

### Stage

One logical part of a pipeline.

### Build

One execution of a Jenkins job or pipeline.

### Agent

The machine where Jenkins executes the pipeline.

### Console Output

The log showing what happened during the build.

### Jenkinsfile

A file containing Jenkins Pipeline code.

---

# 54. Basic Pipeline Structure

```groovy
pipeline {

    agent any

    stages {

        stage('Stage Name') {

            steps {
                // commands
            }

        }

    }
}
```

Important keywords:

```text
pipeline
agent
stages
stage
steps
```

---

# 55. What Is CI?

CI means:

```text
Continuous Integration
```

The basic idea is to frequently integrate code changes and automatically build or test them.

Example:

```text
Developer
    ↓
Git Push
    ↓
GitHub
    ↓
Jenkins
    ↓
Build
    ↓
Test
```

---

# 56. What Is CD?

CD can mean:

```text
Continuous Delivery
```

or:

```text
Continuous Deployment
```

The basic idea is to automate the process of preparing or deploying software.

Our project demonstrates a simple deployment workflow:

```text
GitHub
   ↓
Jenkins
   ↓
Docker
   ↓
AWS EC2
   ↓
Website
```

---

# 57. Troubleshooting

## Jenkins Does Not Open

Check:

```bash
sudo systemctl status jenkins
```

Check:

```bash
sudo ss -lntp | grep 8080
```

Check AWS Security Group port:

```text
8080
```

---

## Jenkins Is Not Running

Run:

```bash
sudo systemctl start jenkins
```

Then:

```bash
sudo systemctl status jenkins
```

For logs:

```bash
sudo journalctl -u jenkins -n 100 --no-pager
```

---

## Jenkins Cannot Use Docker

Test:

```bash
sudo -u jenkins docker ps
```

If permission is denied:

```bash
sudo usermod -aG docker jenkins
```

Restart:

```bash
sudo systemctl restart jenkins
```

Test again:

```bash
sudo -u jenkins docker ps
```

---

## Docker Build Fails

Check:

```bash
docker images
```

Check the project contains:

```text
Dockerfile
index.html
```

Test manually:

```bash
docker build -t jenkins-web-app .
```

---

## Container Is Not Running

Check:

```bash
docker ps -a
```

Check logs:

```bash
docker logs jenkins-web-container
```

---

## Website Does Not Open

Check:

```bash
docker ps
```

Check port:

```bash
sudo ss -lntp | grep 8081
```

Check AWS Security Group port:

```text
8081
```

Then open:

```text
http://EC2-PUBLIC-IP:8081
```

---

## GitHub Clone Fails

Check:

- Repository URL
- Username
- Repository name
- Branch name
- Repository visibility
- Internet connectivity

Test:

```bash
git clone https://github.com/YOUR-USERNAME/jenkins-docker-project.git
```

---

# 58. Final Checklist

## AWS

- [ ] Ubuntu EC2 created
- [ ] SSH works
- [ ] Port 22 configured
- [ ] Port 8080 configured
- [ ] Port 8081 configured

## Jenkins

- [ ] Java installed
- [ ] Jenkins installed
- [ ] Jenkins running
- [ ] Jenkins dashboard opened
- [ ] Admin account created

## Docker

- [ ] Docker installed
- [ ] Docker running
- [ ] `docker run hello-world` works
- [ ] Jenkins can use Docker

## GitHub

- [ ] Repository created
- [ ] `index.html` created
- [ ] `Dockerfile` created
- [ ] Git initialized
- [ ] First commit created
- [ ] `main` branch created
- [ ] Code pushed to GitHub

## Jenkins Pipeline

- [ ] Pipeline created
- [ ] GitHub URL configured
- [ ] Pipeline saved
- [ ] Build #1 completed
- [ ] Console Output checked
- [ ] `Finished: SUCCESS` displayed

## Deployment

- [ ] Docker image created
- [ ] Docker container running
- [ ] Website opened on port 8081
- [ ] Website modified
- [ ] Changes pushed to GitHub
- [ ] Build #2 completed
- [ ] Updated website displayed

---

# 59. Quick Command Reference

## Linux

```bash
sudo apt update
sudo apt upgrade -y
sudo systemctl status jenkins
sudo systemctl restart jenkins
sudo ss -lntp | grep 8080
```

## Git

```bash
git init
git status
git add .
git commit -m "message"
git branch -M main
git remote add origin URL
git push -u origin main
git push
```

## Docker

```bash
docker --version
docker images
docker ps
docker ps -a
docker build -t jenkins-web-app .
docker run -d --name jenkins-web-container -p 8081:80 jenkins-web-app
docker stop jenkins-web-container
docker rm jenkins-web-container
docker logs jenkins-web-container
```

## Jenkins

```text
Jenkins:
http://EC2-PUBLIC-IP:8080
```

```text
Website:
http://EC2-PUBLIC-IP:8081
```

---

# 60. Next Jenkins Topics

After this project, continue in this order:

```text
First Project
     ↓
Freestyle Job
     ↓
Pipeline
     ↓
Jenkinsfile
     ↓
Declarative Pipeline
     ↓
GitHub Webhook
     ↓
Automatic Build on Git Push
     ↓
Jenkins Credentials
     ↓
Environment Variables
     ↓
Build Parameters
     ↓
Docker Hub
     ↓
Jenkins + AWS
     ↓
Complete CI/CD Project
```

---

# Final Architecture

```text
                         DEVELOPER
                             |
                             | git push
                             v
                    +----------------+
                    |     GitHub     |
                    +-------+--------+
                            |
                            | source code
                            v
                    +----------------+
                    |    JENKINS     |
                    |    PIPELINE    |
                    +-------+--------+
                            |
                            | docker build
                            v
                    +----------------+
                    |  DOCKER IMAGE  |
                    +-------+--------+
                            |
                            | docker run
                            v
              +----------------------------+
              |          AWS EC2           |
              |                            |
              |     Docker Container       |
              |          Nginx             |
              |                            |
              +-------------+--------------+
                            |
                            | HTTP :8081
                            v
                    +----------------+
                    |  WEB BROWSER   |
                    +----------------+
```

## Main Flow to Remember

```text
GitHub
   ↓
Jenkins
   ↓
Clone Code
   ↓
Docker Build
   ↓
Stop Old Container
   ↓
Run New Container
   ↓
AWS EC2
   ↓
Web Application
```

**End of Jenkins Beginner Practical Project**
