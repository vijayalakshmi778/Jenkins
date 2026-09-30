# Jenkins Advanced Pipeline Lab
## Jenkins Credentials → Parameters → Environment Variables → Docker Image Tagging → Approval → Deploy → Rollback

### Project Goal

In the previous project, GitHub push automatically triggered Jenkins and Jenkins deployed the Docker application to AWS EC2. fileciteturn0file0L1120-L1150

In this lab, we will make the pipeline more practical by adding Jenkins credentials, parameters, environment variables, image tags, deployment approval, and rollback.

Final flow:

```text
Developer
    |
    | git push
    v
GitHub
    |
    | Webhook
    v
Jenkins
    |
    +--> Checkout
    |
    +--> Test
    |
    +--> Build Docker Image
    |
    +--> Tag Image
    |
    +--> Push Image to Docker Hub
    |
    +--> Manual Approval
    |
    +--> Deploy to AWS EC2
    |
    +--> Health Check
    |
    +--> Rollback if required
```

---

# 1. WHAT YOU ALREADY HAVE

You should already have:

- AWS Ubuntu EC2
- Jenkins
- Docker
- Git
- GitHub repository
- GitHub webhook
- Working Jenkins pipeline
- Docker application
- Application running on EC2

Current process:

```text
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Container
   ↓
AWS EC2
```

---

# 2. WHAT WE WILL ADD

```text
1. Jenkins Credentials
2. Environment Variables
3. Build Parameters
4. Docker Image Tags
5. Docker Hub
6. Automated Test Stage
7. Manual Approval
8. Deployment
9. Health Check
10. Rollback
```

---

# 3. PROJECT FILE STRUCTURE

```text
jenkins-advanced-project/
│
├── index.html
├── Dockerfile
├── Jenkinsfile
└── test.sh
```

---

# 4. CREATE index.html

```html
<!DOCTYPE html>
<html>
<head>
    <title>Jenkins Advanced CI/CD</title>
</head>
<body>
    <h1>Jenkins Advanced Pipeline</h1>
    <p>Version 1</p>
</body>
</html>
```

---

# 5. CREATE Dockerfile

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

---

# 6. CREATE test.sh

```bash
#!/bin/bash

if grep -q "Jenkins Advanced Pipeline" index.html
then
    echo "Test Passed"
    exit 0
else
    echo "Test Failed"
    exit 1
fi
```

On Linux:

```bash
chmod +x test.sh
```

---

# 7. CREATE DOCKER HUB REPOSITORY

Open Docker Hub.

Create:

```text
jenkins-advanced-app
```

Example:

```text
yourdockerusername/jenkins-advanced-app
```

---

# 8. CREATE JENKINS DOCKER HUB CREDENTIAL

Open:

```text
Jenkins
    ↓
Manage Jenkins
    ↓
Credentials
    ↓
System
    ↓
Global credentials
    ↓
Add Credentials
```

Select:

```text
Kind:
Username with password
```

Enter:

```text
Username:
YOUR_DOCKER_USERNAME
```

Password:

```text
Docker Hub Access Token
```

Set ID:

```text
dockerhub-creds
```

Click:

```text
Create
```

---

# 9. VERIFY CREDENTIAL

Go to:

```text
Manage Jenkins
    ↓
Credentials
    ↓
Global
```

Confirm:

```text
dockerhub-creds
```

---

# 10. CREATE JENKINSFILE

Use:

```groovy
pipeline {

    agent any

    environment {
        APP_NAME = 'jenkins-advanced-app'
        IMAGE_NAME = 'YOUR-DOCKER-USERNAME/jenkins-advanced-app'
        CONTAINER_NAME = 'jenkins-advanced-container'
        HOST_PORT = '8081'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/YOUR-GITHUB-USERNAME/jenkins-advanced-project.git'
            }
        }

        stage('Test') {
            steps {
                sh 'chmod +x test.sh'
                sh './test.sh'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Tag Latest') {
            steps {
                sh 'docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest'
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push ${IMAGE_NAME}:${BUILD_NUMBER}'
                sh 'docker push ${IMAGE_NAME}:latest'
            }
        }

        stage('Approval') {
            steps {
                input message: 'Deploy this version to EC2?',
                      ok: 'Deploy'
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    docker pull ${IMAGE_NAME}:${BUILD_NUMBER}

                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      --restart unless-stopped \
                      -p ${HOST_PORT}:80 \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    sleep 5
                    curl -f http://localhost:${HOST_PORT}
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }

        always {
            sh 'docker logout || true'
        }
    }
}
```

---

# 11. CHANGE THE DOCKER USERNAME

Find:

```groovy
IMAGE_NAME = 'YOUR-DOCKER-USERNAME/jenkins-advanced-app'
```

Replace:

```text
YOUR-DOCKER-USERNAME
```

with your Docker Hub username.

Example:

```groovy
IMAGE_NAME = 'vijayalakshmi/jenkins-advanced-app'
```

---

# 12. CHANGE THE GITHUB URL

Find:

```groovy
url: 'https://github.com/YOUR-GITHUB-USERNAME/jenkins-advanced-project.git'
```

Replace it with your repository URL.

---

# 13. BUILD NUMBER

Jenkins automatically creates:

```text
BUILD_NUMBER
```

Example:

```text
Build #1
Build #2
Build #3
```

Docker images become:

```text
jenkins-advanced-app:1
jenkins-advanced-app:2
jenkins-advanced-app:3
```

---

# 14. DOCKER IMAGE TAGGING

For Build #5:

```text
yourdockerusername/jenkins-advanced-app:5
```

and:

```text
yourdockerusername/jenkins-advanced-app:latest
```

The numbered tag is used for versioning and rollback.

---

# 15. COMMIT THE PROJECT

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Add advanced Jenkins pipeline"
```

```bash
git push
```

---

# 16. CONFIGURE JENKINS PIPELINE

Open:

```text
Jenkins
    ↓
Your Project
    ↓
Configure
```

Under Pipeline:

```text
Definition:
Pipeline script from SCM
```

Select:

```text
SCM:
Git
```

Repository:

```text
https://github.com/YOUR-GITHUB-USERNAME/jenkins-advanced-project.git
```

Branch:

```text
*/main
```

Script Path:

```text
Jenkinsfile
```

Save.

---

# 17. TEST BUILD

Click:

```text
Build Now
```

Open:

```text
Console Output
```

Expected stages:

```text
Checkout
Test
Build Docker Image
Tag Latest
Login to Docker Hub
Push Docker Image
Approval
Deploy to EC2
Health Check
```

---

# 18. CHECK TEST

Expected:

```text
Test Passed
```

If:

```text
Test Failed
```

the pipeline stops.

---

# 19. CHECK DOCKER IMAGE

On EC2:

```bash
docker images
```

Example:

```text
REPOSITORY                         TAG
yourusername/jenkins-advanced-app  1
yourusername/jenkins-advanced-app  latest
```

---

# 20. CHECK DOCKER HUB

Open your Docker Hub repository.

You should see:

```text
latest
1
```

After Build #2:

```text
latest
2
1
```

---

# 21. MANUAL APPROVAL

The pipeline stops at:

```text
Approval
```

Jenkins displays:

```text
Deploy this version to EC2?
```

Click:

```text
Deploy
```

---

# 22. DEPLOYMENT

Jenkins executes:

```bash
docker stop jenkins-advanced-container || true
docker rm jenkins-advanced-container || true
docker pull IMAGE:BUILD_NUMBER
docker run ...
```

---

# 23. CHECK CONTAINER

```bash
docker ps
```

Expected:

```text
jenkins-advanced-container
```

---

# 24. CHECK APPLICATION

Open:

```text
http://ELASTIC_IP:8081
```

Expected:

```text
Jenkins Advanced Pipeline
Version 1
```

---

# 25. CREATE VERSION 2

Change:

```html
<p>Version 1</p>
```

to:

```html
<p>Version 2</p>
```

Commit and push:

```bash
git add .
git commit -m "Release version 2"
git push
```

---

# 26. VERSION 2 PIPELINE

GitHub webhook triggers Jenkins.

Jenkins creates:

```text
jenkins-advanced-app:2
```

Then:

```text
Push
 ↓
Approval
 ↓
Deploy
```

Click:

```text
Deploy
```

---

# 27. VERIFY VERSION 2

Open:

```text
http://ELASTIC_IP:8081
```

Expected:

```text
Jenkins Advanced Pipeline
Version 2
```

---

# 28. VIEW ALL IMAGES

```bash
docker images
```

Example:

```text
REPOSITORY                         TAG
yourusername/jenkins-advanced-app  2
yourusername/jenkins-advanced-app  1
yourusername/jenkins-advanced-app  latest
```

---

# 29. ROLLBACK

If Version 2 has a problem:

```text
Version 2
   ↓
Problem
   ↓
Rollback
   ↓
Version 1
```

---

# 30. MANUAL ROLLBACK

```bash
docker stop jenkins-advanced-container || true
docker rm jenkins-advanced-container || true
```

Pull Version 1:

```bash
docker pull YOUR-DOCKER-USERNAME/jenkins-advanced-app:1
```

Run Version 1:

```bash
docker run -d \
  --name jenkins-advanced-container \
  --restart unless-stopped \
  -p 8081:80 \
  YOUR-DOCKER-USERNAME/jenkins-advanced-app:1
```

---

# 31. VERIFY ROLLBACK

```bash
docker ps
```

Open:

```text
http://ELASTIC_IP:8081
```

Version 1 should be running.

---

# 32. ADD JENKINS PARAMETERS

Add inside `pipeline`:

```groovy
parameters {
    choice(
        name: 'DEPLOY_VERSION',
        choices: ['latest', '1', '2'],
        description: 'Select Docker image version'
    )
}
```

The beginning becomes:

```groovy
pipeline {

    agent any

    parameters {
        choice(
            name: 'DEPLOY_VERSION',
            choices: ['latest', '1', '2'],
            description: 'Select Docker image version'
        )
    }

    environment {
        APP_NAME = 'jenkins-advanced-app'
        IMAGE_NAME = 'YOUR-DOCKER-USERNAME/jenkins-advanced-app'
        CONTAINER_NAME = 'jenkins-advanced-container'
        HOST_PORT = '8081'
    }
```

---

# 33. USE DEPLOY_VERSION

Change deployment to:

```bash
docker pull ${IMAGE_NAME}:${DEPLOY_VERSION}
```

and:

```bash
docker run -d \
  --name ${CONTAINER_NAME} \
  --restart unless-stopped \
  -p ${HOST_PORT}:80 \
  ${IMAGE_NAME}:${DEPLOY_VERSION}
```

---

# 34. RUN WITH PARAMETERS

Open:

```text
Jenkins
    ↓
Build with Parameters
```

Select:

```text
DEPLOY_VERSION = 1
```

Click:

```text
Build
```

---

# 35. DEPLOY VERSION 1

```text
Build with Parameters
        ↓
DEPLOY_VERSION=1
        ↓
Docker Pull :1
        ↓
Deploy
```

---

# 36. DEPLOY VERSION 2

Run:

```text
Build with Parameters
```

Select:

```text
DEPLOY_VERSION = 2
```

Click:

```text
Build
```

---

# 37. ENVIRONMENT VARIABLES

Add:

```groovy
stage('Show Environment') {
    steps {
        sh 'echo "Build Number: ${BUILD_NUMBER}"'
        sh 'echo "Job Name: ${JOB_NAME}"'
        sh 'echo "Workspace: ${WORKSPACE}"'
        sh 'echo "Deploy Version: ${DEPLOY_VERSION}"'
    }
}
```

Do not print passwords or tokens.

---

# 38. DEPLOYMENT INFORMATION

Add:

```groovy
stage('Deployment Information') {
    steps {
        echo "Application: ${APP_NAME}"
        echo "Image: ${IMAGE_NAME}"
        echo "Version: ${DEPLOY_VERSION}"
        echo "Build: ${BUILD_NUMBER}"
    }
}
```

---

# 39. FINAL PIPELINE

```text
GitHub
   |
   | Webhook
   v
Jenkins
   |
   +--> Checkout
   |
   +--> Test
   |
   +--> Build Image
   |
   +--> Tag Image
   |
   +--> Docker Hub Login
   |
   +--> Push Image
   |
   +--> Approval
   |
   +--> Deploy
   |
   +--> Health Check
   |
   v
AWS EC2
```

---

# 40. FINAL IMAGE FLOW

```text
Source Code
    ↓
Jenkins
    ↓
Docker Build
    ↓
Image :1
    ↓
Docker Hub

Source Code
    ↓
Jenkins
    ↓
Docker Build
    ↓
Image :2
    ↓
Docker Hub
```

---

# 41. ROLLBACK FLOW

```text
Version 2
    ↓
Production Problem
    ↓
Build with Parameters
    ↓
DEPLOY_VERSION=1
    ↓
Docker Pull :1
    ↓
Deploy :1
```

---

# 42. FINAL PROJECT ARCHITECTURE

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
                       |    :8080    |
                       +------+------+
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          Checkout          Test        Build Image
                                             |
                                             v
                                      Docker Image
                                      :BUILD_NUMBER
                                             |
                                             v
                                       Docker Hub
                                             |
                                             v
                                         Approval
                                             |
                                             v
                                      AWS EC2
                                             |
                                             v
                                    Docker Container
                                             |
                                             v
                                      Web Application
```

---

# 43. JENKINS CONCEPTS COMPLETED

```text
✓ Jenkins Pipeline
✓ Jenkinsfile
✓ Jenkins Credentials
✓ Credentials Binding
✓ Environment Variables
✓ Build Parameters
✓ BUILD_NUMBER
✓ Docker Image Tagging
✓ Docker Hub
✓ Automated Testing
✓ Manual Approval
✓ Deployment
✓ Health Check
✓ Rollback
```

---

# 44. STUDENT PRACTICAL TASK

```text
[ ] Create GitHub repository
[ ] Create Dockerfile
[ ] Create test.sh
[ ] Create Jenkinsfile
[ ] Create Docker Hub repository
[ ] Add Docker Hub credential to Jenkins
[ ] Build Docker image
[ ] Tag image using BUILD_NUMBER
[ ] Push image to Docker Hub
[ ] Configure GitHub webhook
[ ] Add manual approval
[ ] Deploy to EC2
[ ] Perform health check
[ ] Create Version 2
[ ] Deploy Version 2
[ ] Roll back to Version 1
[ ] Use Build with Parameters
```

---

# 45. STUDENT CHALLENGE

Modify the pipeline to contain:

```text
Stage 1 → Checkout
Stage 2 → Test
Stage 3 → Build
Stage 4 → Tag
Stage 5 → Push
Stage 6 → Approval
Stage 7 → Deploy
Stage 8 → Health Check
Stage 9 → Cleanup
```

Add:

```groovy
stage('Cleanup') {
    steps {
        sh 'docker image prune -f'
    }
}
```

---

# 46. FINAL TEST

Change:

```html
<h1>DevOps Students - Jenkins CI/CD</h1>
```

Run:

```bash
git add .
git commit -m "Update application"
git push
```

Verify:

```text
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Test
   ↓
Docker Build
   ↓
Docker Hub
   ↓
Approval
   ↓
EC2
   ↓
Application
```

Open:

```text
http://ELASTIC_IP:8081
```

---

# 47. FINAL RESULT

```text
GitHub
   ↓
Jenkins Webhook
   ↓
Jenkins Pipeline
   ↓
Automated Test
   ↓
Docker Build
   ↓
Docker Image Version
   ↓
Docker Hub
   ↓
Manual Approval
   ↓
AWS EC2 Deployment
   ↓
Health Check
   ↓
Rollback
```

# END OF LAB
