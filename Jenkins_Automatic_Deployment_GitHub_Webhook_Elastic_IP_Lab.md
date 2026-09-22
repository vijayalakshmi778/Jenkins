# Jenkins Automatic Deployment Lab
## GitHub Webhook → Jenkins → Docker → AWS EC2

### Project Goal

In the first Jenkins project, deployment was started manually by clicking **Build Now**.

In this lab, we will change that process so that a code push to GitHub automatically triggers Jenkins.

Final flow:

```text
Developer
    |
    | git push
    v
GitHub Repository
    |
    | Webhook
    v
Jenkins
    |
    | Clone latest code
    | Build Docker image
    | Stop old container
    | Run new container
    v
Docker Container
    |
    v
AWS EC2
    |
    v
Updated Web Application
```

---

# 1. WHAT YOU ALREADY HAVE

This lab assumes the first Jenkins project is already working.

You should already have:

- AWS Ubuntu EC2 server
- Jenkins installed
- Docker installed
- Git installed
- Jenkins able to use Docker
- GitHub repository
- `index.html`
- `Dockerfile`
- Jenkins pipeline
- Application running on port `8081`
- Jenkins running on port `8080`

Current architecture:

```text
AWS EC2
│
├── Jenkins → Port 8080
│
└── Docker Application → Port 8081
```

---

# 2. WHY WE NEED AN ELASTIC IP

A normal EC2 public IPv4 address can change when an instance is stopped and started.

Example:

```text
Before stopping EC2:

Public IP = 13.XX.XX.10
```

After starting EC2:

```text
Public IP = 13.XX.XX.25
```

This becomes a problem when GitHub Webhook is configured.

For example, if GitHub sends the webhook to:

```text
http://13.XX.XX.10:8080/github-webhook/
```

and the EC2 server later receives:

```text
13.XX.XX.25
```

GitHub will still try to contact the old address.

Therefore, use an AWS Elastic IP for this lab.

---

# 3. ELASTIC IP CONCEPT

An Elastic IP gives the EC2 server a stable public IPv4 address.

Architecture:

```text
                 Elastic IP
                 13.XX.XX.50
                      |
                      v
              +---------------+
              |    AWS EC2    |
              |               |
              |    Jenkins    |
              |    :8080      |
              |               |
              |    Docker     |
              |    :8081      |
              +---------------+
```

After using an Elastic IP:

```text
Stop EC2
   ↓
Start EC2
   ↓
Elastic IP remains associated
   ↓
Jenkins URL remains the same
   ↓
GitHub Webhook can continue using the same address
```

---

# 4. IMPORTANT AWS COST NOTE

Elastic IP addresses can incur AWS charges when they are allocated but not associated with an eligible resource, and AWS pricing rules can change.

For a learning lab:

- Use the Elastic IP while the project is active.
- Release unused Elastic IPs when you are completely finished with the lab.
- Check the current AWS pricing page before leaving resources running.

---

# 5. CREATE AN ELASTIC IP

Open:

```text
AWS Console
    ↓
EC2
    ↓
Elastic IPs
```

Click:

```text
Allocate Elastic IP address
```

Keep:

```text
Public IPv4 address pool
Amazon's pool of IPv4 addresses
```

Click:

```text
Allocate
```

You will receive an address similar to:

```text
13.XX.XX.50
```

This is your Elastic IP.

---

# 6. ASSOCIATE ELASTIC IP WITH EC2

Select the newly created Elastic IP.

Click:

```text
Actions
    ↓
Associate Elastic IP address
```

For:

```text
Resource type
```

select:

```text
Instance
```

Select your Jenkins EC2 instance.

Click:

```text
Associate
```

---

# 7. VERIFY THE ELASTIC IP

Go to:

```text
EC2
    ↓
Instances
```

Select your Jenkins server.

Find:

```text
Public IPv4 address
```

It should show your Elastic IP.

Example:

```text
13.XX.XX.50
```

From this point onward, call this:

```text
ELASTIC_IP
```

Do not use the old temporary public IP in the webhook configuration.

---

# 8. TEST JENKINS USING ELASTIC IP

Open:

```text
http://ELASTIC_IP:8080
```

Example:

```text
http://13.XX.XX.50:8080
```

Jenkins should open.

---

# 9. TEST THE APPLICATION

Open:

```text
http://ELASTIC_IP:8081
```

Example:

```text
http://13.XX.XX.50:8081
```

The application should open.

---

# 10. TEST STOP AND START

This is an important test.

In AWS:

```text
EC2
    ↓
Select Instance
    ↓
Instance state
    ↓
Stop instance
```

Wait until:

```text
Stopped
```

Then:

```text
Start instance
```

Wait until:

```text
Running
```

Check the IP again.

Your Elastic IP should remain the same.

Example:

```text
Before:

13.XX.XX.50

After stop/start:

13.XX.XX.50
```

This is why we use an Elastic IP.

---

# 11. CONNECT TO EC2 AFTER RESTART

From Windows PowerShell:

```powershell
ssh -i "your-key.pem" ubuntu@ELASTIC_IP
```

Example:

```powershell
ssh -i "mykey.pem" ubuntu@13.XX.XX.50
```

---

# 12. CHECK JENKINS

Run:

```bash
sudo systemctl status jenkins
```

Expected:

```text
active (running)
```

Press:

```text
q
```

---

# 13. CHECK DOCKER

Run:

```bash
docker --version
```

Then:

```bash
docker ps
```

Check whether the application container is running.

---

# 14. MAKE THE CONTAINER RESTART AUTOMATICALLY

For a production-style lab, the container can use Docker's restart policy.

If your current container is running, first check:

```bash
docker ps
```

Stop and remove the old container if you want to recreate it:

```bash
docker stop jenkins-web-container
```

```bash
docker rm jenkins-web-container
```

Then run:

```bash
docker run -d \
  --name jenkins-web-container \
  --restart unless-stopped \
  -p 8081:80 \
  jenkins-web-app
```

Check:

```bash
docker ps
```

The container should be running.

---

# 15. UNDERSTAND `--restart unless-stopped`

This option:

```text
--restart unless-stopped
```

tells Docker to automatically restart the container when the Docker service/server comes back up, unless the container was intentionally stopped.

This helps when the EC2 instance is restarted.

---

# 16. PROJECT FILE STRUCTURE

On your Windows computer, your project should now contain:

```text
jenkins-docker-project/
│
├── index.html
├── Dockerfile
└── Jenkinsfile
```

---

# 17. CREATE JENKINSFILE

Open the project in VS Code.

Create:

```text
Jenkinsfile
```

Make sure it is exactly:

```text
Jenkinsfile
```

Not:

```text
Jenkinsfile.txt
```

---

# 18. JENKINSFILE

Use the following pipeline:

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
                sh 'docker run -d --name jenkins-web-container --restart unless-stopped -p 8081:80 jenkins-web-app'
            }
        }
    }
}
```

---

# 19. CHANGE YOUR GITHUB URL

Find:

```groovy
url: 'https://github.com/YOUR-USERNAME/jenkins-docker-project.git'
```

Replace:

```text
YOUR-USERNAME
```

with your actual GitHub username.

Example:

```groovy
url: 'https://github.com/student123/jenkins-docker-project.git'
```

---

# 20. UNDERSTAND THE JENKINSFILE

## Pipeline

```groovy
pipeline {
```

Defines the Jenkins pipeline.

---

## Agent

```groovy
agent any
```

Allows Jenkins to execute the pipeline on an available Jenkins agent.

---

## Clone Stage

```groovy
stage('Clone GitHub Repository')
```

Jenkins downloads the latest code from GitHub.

---

## Build Stage

```groovy
docker build -t jenkins-web-app .
```

Creates a new Docker image using the latest source code.

---

## Stop Stage

```bash
docker stop jenkins-web-container || true
```

Attempts to stop the old container.

`|| true` prevents the pipeline from failing when the container does not exist.

---

## Remove Stage

```bash
docker rm jenkins-web-container || true
```

Removes the old container.

---

## Run Stage

```bash
docker run -d --name jenkins-web-container --restart unless-stopped -p 8081:80 jenkins-web-app
```

Starts the new version.

---

# 21. SAVE THE JENKINSFILE

Save the file.

Your project:

```text
jenkins-docker-project/
│
├── index.html
├── Dockerfile
└── Jenkinsfile
```

---

# 22. TEST THE PROJECT LOCALLY

Open PowerShell in the project folder.

Run:

```powershell
dir
```

You should see:

```text
index.html
Dockerfile
Jenkinsfile
```

---

# 23. COMMIT JENKINSFILE TO GITHUB

Check Git:

```bash
git status
```

Add the file:

```bash
git add Jenkinsfile
```

Commit:

```bash
git commit -m "Add Jenkins pipeline"
```

Push:

```bash
git push
```

Refresh GitHub.

You should now see:

```text
index.html
Dockerfile
Jenkinsfile
```

---

# 24. CONFIGURE JENKINS TO USE JENKINSFILE

Open:

```text
http://ELASTIC_IP:8080
```

Open your Jenkins project.

Click:

```text
Configure
```

Find:

```text
Pipeline
```

Under:

```text
Definition
```

select:

```text
Pipeline script from SCM
```

---

# 25. SELECT SCM

Select:

```text
Git
```

Repository URL:

```text
https://github.com/YOUR-USERNAME/jenkins-docker-project.git
```

Example:

```text
https://github.com/student123/jenkins-docker-project.git
```

Branch:

```text
*/main
```

Script Path:

```text
Jenkinsfile
```

Save the configuration.

---

# 26. TEST THE JENKINSFILE

Click:

```text
Build Now
```

Open:

```text
Build
    ↓
Console Output
```

The pipeline should show stages similar to:

```text
Clone GitHub Repository
Build Docker Image
Stop Old Container
Run New Container
```

At the end:

```text
Finished: SUCCESS
```

---

# 27. IMPORTANT: WHY WE FIRST TEST BUILD NOW

At this point, we are testing the Jenkinsfile itself.

We are NOT testing the webhook yet.

First confirm:

```text
Jenkins
   ↓
Jenkinsfile
   ↓
GitHub
   ↓
Docker
   ↓
Application
```

works correctly.

Only after that should we configure automatic triggering.

---

# 28. CONFIGURE JENKINS WEBHOOK TRIGGER

Open your Jenkins project.

Click:

```text
Configure
```

Find:

```text
Build Triggers
```

Enable:

```text
GitHub hook trigger for GITScm polling
```

The exact wording can vary slightly depending on Jenkins/plugin versions.

Click:

```text
Save
```

---

# 29. UNDERSTAND THE WEBHOOK

A webhook is a notification sent by GitHub to Jenkins.

Without webhook:

```text
Developer
   ↓
git push
   ↓
GitHub
   ↓
Developer manually clicks Build Now
```

With webhook:

```text
Developer
   ↓
git push
   ↓
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Pipeline automatically starts
```

---

# 30. CREATE GITHUB WEBHOOK

Open your GitHub repository.

Go to:

```text
Settings
    ↓
Webhooks
    ↓
Add webhook
```

---

# 31. WEBHOOK PAYLOAD URL

Set:

```text
Payload URL
```

to:

```text
http://ELASTIC_IP:8080/github-webhook/
```

Example:

```text
http://13.XX.XX.50:8080/github-webhook/
```

IMPORTANT:

Use the Elastic IP.

Do NOT use:

```text
http://OLD-EC2-PUBLIC-IP:8080/github-webhook/
```

---

# 32. CONTENT TYPE

For:

```text
Content type
```

select:

```text
application/json
```

---

# 33. WHICH EVENTS?

Select:

```text
Just the push event
```

This means GitHub will notify Jenkins when code is pushed.

---

# 34. ACTIVE

Make sure:

```text
☑ Active
```

is enabled.

Click:

```text
Add webhook
```

---

# 35. GITHUB WEBHOOK TEST

GitHub may send a test request after the webhook is created.

Go to:

```text
Repository
    ↓
Settings
    ↓
Webhooks
    ↓
Your webhook
```

Open the webhook.

Check:

```text
Recent Deliveries
```

You should see the delivery request.

---

# 36. TEST AUTOMATIC DEPLOYMENT

Now we will perform the actual test.

Open:

```text
index.html
```

Change the website text.

For example, change:

```html
<h1>Jenkins Deployment Successful!</h1>
```

to:

```html
<h1>Jenkins Automatic Deployment Successful!</h1>
```

Save the file.

---

# 37. CHECK GIT STATUS

In PowerShell:

```bash
git status
```

You should see:

```text
modified: index.html
```

---

# 38. ADD THE CHANGE

Run:

```bash
git add .
```

---

# 39. COMMIT THE CHANGE

Run:

```bash
git commit -m "Update website"
```

---

# 40. PUSH TO GITHUB

Run:

```bash
git push
```

This is the only action you need to perform for deployment.

You should NOT click:

```text
Build Now
```

---

# 41. WHAT HAPPENS NOW?

The complete process is:

```text
VS Code
   |
   | git push
   v
GitHub
   |
   | Webhook
   v
Jenkins
   |
   | Get latest code
   v
Docker Build
   |
   | New image
   v
Stop old container
   |
   v
Remove old container
   |
   v
Run new container
   |
   v
AWS EC2 :8081
```

---

# 42. WATCH JENKINS

Open:

```text
http://ELASTIC_IP:8080
```

Open the Jenkins project.

A new build should start automatically.

You should see a new build number:

```text
Build #2
```

or:

```text
Build #3
```

depending on your previous builds.

---

# 43. CHECK CONSOLE OUTPUT

Open the new build.

Click:

```text
Console Output
```

You should see stages such as:

```text
Clone GitHub Repository
```

Then:

```text
Build Docker Image
```

Then:

```text
Stop Old Container
```

Then:

```text
Run New Container
```

At the end:

```text
Finished: SUCCESS
```

---

# 44. CHECK THE WEBSITE

Open:

```text
http://ELASTIC_IP:8081
```

Example:

```text
http://13.XX.XX.50:8081
```

Refresh the page.

The updated website should appear.

---

# 45. THE IMPORTANT RESULT

You changed:

```text
index.html
```

You pushed:

```bash
git push
```

You did NOT:

```text
SSH into EC2
```

You did NOT:

```text
docker build manually
```

You did NOT:

```text
docker stop manually
```

You did NOT:

```text
docker run manually
```

You did NOT:

```text
Jenkins → Build Now
```

Jenkins performed the deployment automatically.

---

# 46. FINAL AUTOMATIC DEPLOYMENT FLOW

Your final architecture is:

```text
                    DEVELOPER
                        |
                        | git push
                        v
                 +-------------+
                 |   GitHub    |
                 +------+------+
                        |
                        | Webhook
                        v
                 +-------------+
                 |   Jenkins   |
                 |   :8080     |
                 +------+------+
                        |
                        | Jenkinsfile
                        v
                 +-------------+
                 | Docker Build|
                 +------+------+
                        |
                        v
                 +-------------+
                 |Docker Image |
                 +------+------+
                        |
                        v
                 +-------------+
                 |Docker       |
                 |Container    |
                 |Nginx :80    |
                 +------+------+
                        |
                        | Port 8081
                        v
                 +-------------+
                 |   AWS EC2   |
                 | Elastic IP  |
                 +------+------+
                        |
                        v
                  WEB BROWSER
```

---

# 47. PORT ARCHITECTURE

Your server uses:

```text
Port 22
    ↓
SSH

Port 8080
    ↓
Jenkins

Port 8081
    ↓
Docker Web Application
```

Example:

```text
SSH:
ELASTIC_IP:22

Jenkins:
ELASTIC_IP:8080

Application:
ELASTIC_IP:8081
```

---

# 48. AWS SECURITY GROUP

Make sure the Security Group allows the required ports.

Example:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH |
| 8080 | TCP | Jenkins |
| 8081 | TCP | Web Application |

For a classroom lab, you can configure access according to your testing needs.

For real deployments, restrict access instead of unnecessarily opening ports to the whole internet.

---

# 49. IMPORTANT SECURITY NOTE

Do not put:

```text
AWS Access Keys
Passwords
API Keys
Tokens
Private Keys
```

inside:

```text
GitHub
Jenkinsfile
Dockerfile
```

Use Jenkins Credentials for secrets when credentials are required.

---

# 50. WHAT HAPPENS WHEN YOU STOP EC2?

This is now your normal workflow:

```text
Finish work
    ↓
Stop EC2
```

Later:

```text
Start EC2
    ↓
Elastic IP remains stable
    ↓
Jenkins starts
    ↓
Docker starts
    ↓
Application can restart
```

Check:

```bash
sudo systemctl status jenkins
```

Then:

```bash
docker ps
```

---

# 51. WHAT HAPPENS TO GITHUB WEBHOOK?

Because the webhook uses:

```text
Elastic IP
```

instead of a temporary EC2 public IP, you do not need to change the webhook every time the EC2 instance is stopped and started.

Webhook:

```text
http://ELASTIC_IP:8080/github-webhook/
```

This remains your stable Jenkins endpoint while the Elastic IP is associated with the server.

---

# 52. IF JENKINS DOES NOT START AFTER REBOOT

Check:

```bash
sudo systemctl status jenkins
```

If necessary:

```bash
sudo systemctl start jenkins
```

Make sure Jenkins is enabled:

```bash
sudo systemctl enable jenkins
```

---

# 53. IF DOCKER CONTAINER IS NOT RUNNING

Check:

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

Check logs:

```bash
docker logs jenkins-web-container
```

If necessary, the Jenkins pipeline can redeploy the application.

---

# 54. IF WEBHOOK DOES NOT TRIGGER JENKINS

Check these in order:

### Check 1 — Jenkins is running

```bash
sudo systemctl status jenkins
```

### Check 2 — Port 8080 is listening

```bash
sudo ss -lntp | grep 8080
```

### Check 3 — AWS Security Group

Make sure port `8080` is allowed.

### Check 4 — Jenkins trigger

Go to:

```text
Jenkins
    ↓
Project
    ↓
Configure
    ↓
Build Triggers
```

Make sure:

```text
GitHub hook trigger for GITScm polling
```

is enabled.

### Check 5 — GitHub webhook URL

It should be:

```text
http://ELASTIC_IP:8080/github-webhook/
```

### Check 6 — GitHub delivery

Go to:

```text
GitHub
    ↓
Settings
    ↓
Webhooks
    ↓
Your webhook
    ↓
Recent Deliveries
```

Check the delivery result.

---

# 55. IF GITHUB CLONE FAILS

Check:

```text
GitHub repository URL
```

Check:

```text
Branch = main
```

Check:

```text
Jenkinsfile
```

Check repository access.

For a private repository, configure Jenkins credentials rather than putting a password/token directly in the Jenkinsfile.

---

# 56. IF DOCKER BUILD FAILS

SSH into EC2.

Check:

```bash
ls
```

The Jenkins workspace should contain:

```text
index.html
Dockerfile
Jenkinsfile
```

Check Docker manually:

```bash
docker build -t jenkins-web-app .
```

---

# 57. IF PORT 8081 IS ALREADY IN USE

Check:

```bash
sudo ss -lntp | grep 8081
```

Also:

```bash
docker ps
```

Find which container is using the port.

---

# 58. IF JENKINS CANNOT ACCESS DOCKER

Run:

```bash
sudo -u jenkins docker ps
```

If permission is denied:

```bash
sudo usermod -aG docker jenkins
```

Then:

```bash
sudo systemctl restart jenkins
```

Test again:

```bash
sudo -u jenkins docker ps
```

---

# 59. COMPLETE TEST

Perform this complete test from beginning to end.

### Step 1

Start EC2.

### Step 2

Confirm Elastic IP.

### Step 3

Open:

```text
http://ELASTIC_IP:8080
```

### Step 4

Confirm Jenkins is running.

### Step 5

Confirm Docker is running.

### Step 6

Open:

```text
http://ELASTIC_IP:8081
```

### Step 7

Change `index.html`.

### Step 8

Run:

```bash
git add .
```

### Step 9

Run:

```bash
git commit -m "Test automatic deployment"
```

### Step 10

Run:

```bash
git push
```

### Step 11

Do NOT click:

```text
Build Now
```

### Step 12

Open Jenkins.

### Step 13

Check whether a new build started.

### Step 14

Open Console Output.

### Step 15

Confirm:

```text
Finished: SUCCESS
```

### Step 16

Open:

```text
http://ELASTIC_IP:8081
```

### Step 17

Confirm the new website version.

---

# 60. FINAL CI/CD FLOW

Your project now demonstrates:

```text
                CODE
                 |
                 v
          +-------------+
          |   GitHub    |
          +------+------+
                 |
                 | Webhook
                 v
          +-------------+
          |   Jenkins   |
          +------+------+
                 |
                 v
          +-------------+
          | Docker Build|
          +------+------+
                 |
                 v
          +-------------+
          |Docker Image |
          +------+------+
                 |
                 v
          +-------------+
          |   Docker    |
          |  Container  |
          +------+------+
                 |
                 v
          +-------------+
          |   AWS EC2   |
          | Elastic IP  |
          +------+------+
                 |
                 v
          Web Application
```

---

# 61. WHAT YOU HAVE LEARNED

After completing this lab, you understand:

- AWS EC2
- Elastic IP
- Jenkins
- Jenkins Pipeline
- Jenkinsfile
- Git
- GitHub
- GitHub Webhook
- Docker
- Docker Image
- Docker Container
- Automatic deployment
- CI/CD basics

---

# 62. OLD PROCESS VS NEW PROCESS

## Before

```text
Developer
   ↓
git push
   ↓
GitHub
   ↓
Open Jenkins
   ↓
Build Now
   ↓
Docker Build
   ↓
Docker Run
   ↓
Application
```

## After

```text
Developer
   ↓
git push
   ↓
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Run
   ↓
Application
```

The major improvement is:

```text
Build Now
```

is no longer required.

---

# 63. FINAL PROJECT ARCHITECTURE

```text
                         INTERNET
                             |
                             v
                    +----------------+
                    |   GitHub       |
                    |   Repository   |
                    +-------+--------+
                            |
                            | Webhook
                            v
                    +----------------+
                    |    Jenkins     |
                    |    :8080       |
                    +-------+--------+
                            |
                            | Pipeline
                            v
                    +----------------+
                    | Docker Build   |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    | Docker Image    |
                    +-------+--------+
                            |
                            v
              +-----------------------------+
              |          AWS EC2             |
              |                             |
              |  Elastic IP                  |
              |                             |
              |  Jenkins :8080              |
              |                             |
              |  Docker Container           |
              |  Nginx :80                  |
              |       |                     |
              |       +---- Host :8081      |
              +-------------+---------------+
                            |
                            v
                       Web Browser
```

---

# 64. FINAL CHECKLIST

## AWS

- [ ] Ubuntu EC2 created
- [ ] Elastic IP allocated
- [ ] Elastic IP associated with EC2
- [ ] Port 22 configured
- [ ] Port 8080 configured
- [ ] Port 8081 configured

## Jenkins

- [ ] Jenkins installed
- [ ] Jenkins running
- [ ] Jenkins enabled at boot
- [ ] Jenkins can access Docker
- [ ] Jenkinsfile created
- [ ] Pipeline tested successfully

## Docker

- [ ] Docker installed
- [ ] Docker service running
- [ ] Docker image created
- [ ] Docker container running
- [ ] Restart policy configured

## GitHub

- [ ] Repository created
- [ ] `index.html` pushed
- [ ] `Dockerfile` pushed
- [ ] `Jenkinsfile` pushed
- [ ] Webhook created
- [ ] Webhook uses Elastic IP
- [ ] Push event enabled

## Automatic Deployment

- [ ] Code changed
- [ ] `git add .`
- [ ] `git commit`
- [ ] `git push`
- [ ] Jenkins automatically triggered
- [ ] Docker image rebuilt
- [ ] Old container replaced
- [ ] New application deployed
- [ ] Website updated

---

# 65. FINAL RESULT

The final development process is:

```text
Developer changes code
        ↓
git push
        ↓
GitHub
        ↓
GitHub Webhook
        ↓
Jenkins
        ↓
Jenkinsfile
        ↓
Docker Build
        ↓
New Docker Image
        ↓
Stop Old Container
        ↓
Run New Container
        ↓
AWS EC2
        ↓
Updated Website
```

The developer only needs to:

```bash
git add .
git commit -m "Update application"
git push
```

Jenkins handles the deployment automatically.

---

# 66. NEXT JENKINS TOPICS

After successfully completing this project, continue with:

```text
1. Jenkins Credentials
2. Private GitHub Repository
3. GitHub Personal Access Token / SSH authentication
4. Jenkins Environment Variables
5. Build Parameters
6. Docker Image Tagging
7. Docker Hub
8. Jenkins + Docker Hub
9. Automated Testing
10. Multi-stage Pipeline
11. CI/CD Pipeline
12. Jenkins + AWS Deployment
13. Production-style deployment
```

# END OF LAB
