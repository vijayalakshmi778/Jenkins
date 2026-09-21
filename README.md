JENKINS PRACTICAL LAB
==============================

Project: GitHub → Jenkins → Docker → AWS EC2 → Web Application

This is the first Jenkins project for beginners.

The project connects the topics already learned:

Linux → AWS EC2 → Git → GitHub → Docker → Jenkins

The final flow will be:

Developer
   |
   v
GitHub Repository
   |
   v
Jenkins
   |
   v
Clone/Pull Code
   |
   v
Build Docker Image
   |
   v
Stop Old Container
   |
   v
Run New Container
   |
   v
AWS EC2
   |
   v
Web Application


1. WHAT IS JENKINS?
===================

Jenkins is an automation server.

In a normal deployment, a developer may manually:

1. Get the latest code from GitHub.
2. Build the application.
3. Build a Docker image.
4. Stop the old container.
5. Start the new container.
6. Check whether the application is working.

Jenkins can automate these steps.

For this lab, Jenkins will:

GitHub
   ↓
Get the latest code
   ↓
Build Docker image
   ↓
Remove old container
   ↓
Run new Docker container
   ↓
Application available on EC2


2. WHAT YOU WILL BUILD
======================

You will create a simple HTML website.

The website will be stored in GitHub.

Jenkins will run on an Ubuntu AWS EC2 server.

Jenkins will execute a pipeline that:

1. Gets the website code from GitHub.
2. Builds a Docker image.
3. Stops/removes the previous container if it exists.
4. Starts a new Docker container.
5. Publishes the website on port 8080.

Final URL:

http://EC2-PUBLIC-IP:8080


3. LAB REQUIREMENTS
===================

Before starting Jenkins, make sure you already have:

- AWS account
- Ubuntu EC2 instance
- Basic Linux commands
- Git basics
- GitHub repository
- Docker basics
- Basic AWS Security Group knowledge

For this lab, use one Ubuntu EC2 server for Jenkins and Docker.

Example:

Ubuntu EC2
Public IP: 13.XX.XX.XX

You can use your own EC2 public IP.


4. AWS EC2 REQUIREMENTS
=======================

Create or use an Ubuntu EC2 instance.

Recommended for a beginner lab:

Instance type:
t2.micro or t3.micro, depending on AWS availability and your account.

Security Group inbound rules:

SSH
Port: 22
Source: My IP

Jenkins
Port: 8080
Source: My IP

Application
Port: 8080
Source: 0.0.0.0/0

IMPORTANT:

Port 8080 is used by both Jenkins and the application in this beginner lab.

Therefore, do NOT run Jenkins and the application on the same port.

To avoid this conflict, use:

Jenkins:
Port 8080

Application:
Port 8081

Final application URL:

http://EC2-PUBLIC-IP:8081

Jenkins URL:

http://EC2-PUBLIC-IP:8080


5. CONNECT TO UBUNTU EC2
=========================

From Windows PowerShell:

ssh -i "your-key.pem" ubuntu@EC2-PUBLIC-IP

Example:

ssh -i "mykey.pem" ubuntu@13.XX.XX.XX

If the key permission causes a problem on Windows, use the appropriate Windows/OpenSSH permissions for your .pem file.

After successful login:

ubuntu@ip-172-31-xx-xx:~$


6. UPDATE UBUNTU
================

Run:

sudo apt update

Then:

sudo apt upgrade -y

Explanation:

apt update
Downloads the latest package information.

apt upgrade
Installs available updates.


7. CHECK JAVA
=============

Jenkins requires Java.

Check whether Java is already installed:

java -version

If Java is not installed, install OpenJDK 21:

sudo apt install fontconfig openjdk-21-jre -y

Check again:

java -version

You should see Java version information.


8. INSTALL JENKINS
==================

Jenkins provides an official Debian/Ubuntu installation method.

Install the Jenkins repository key:

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

Add the Jenkins repository:

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null

Update package information:

sudo apt update

Install Jenkins:

sudo apt install jenkins -y


9. START JENKINS
================

Start Jenkins:

sudo systemctl start jenkins

Enable Jenkins automatically after server reboot:

sudo systemctl enable jenkins

Check Jenkins status:

sudo systemctl status jenkins

You should see:

active (running)

Press:

q

to exit the status screen.


10. CHECK JENKINS PORT
=====================

Run:

sudo ss -lntp | grep 8080

Jenkins normally listens on:

8080


11. OPEN JENKINS IN BROWSER
===========================

Open:

http://EC2-PUBLIC-IP:8080

Example:

http://13.XX.XX.XX:8080

You should see the Jenkins setup screen.


12. GET THE INITIAL JENKINS PASSWORD
====================================

Run:

sudo cat /var/lib/jenkins/secrets/initialAdminPassword

Copy the password.

Paste it into the Jenkins browser page.

Click:

Continue


13. INSTALL JENKINS PLUGINS
===========================

Jenkins will show plugin installation options.

For the first lab:

Select:

Install suggested plugins

Wait for the installation to finish.

Do not close the browser during installation.


14. CREATE JENKINS ADMIN USER
=============================

Create an administrator account.

Example:

Username:
jenkinsadmin

Password:
Create your own password

Full name:
Your Name

Email:
Your email address

Click:

Save and Continue


15. JENKINS URL
===============

Jenkins will show the Jenkins URL.

Keep the default URL:

http://EC2-PUBLIC-IP:8080/

Click:

Save and Finish

Then click:

Start using Jenkins


16. VERIFY JENKINS
==================

You should now see the Jenkins dashboard.

Jenkins dashboard is the main page from where you create and manage:

- Jobs
- Pipelines
- Builds
- Credentials
- Plugins


17. INSTALL GIT
===============

Jenkins will need Git to download code from GitHub.

Check Git:

git --version

If Git is not installed:

sudo apt install git -y

Check:

git --version


18. INSTALL DOCKER
==================

The Jenkins pipeline will build and run Docker containers.

Check Docker:

docker --version

If Docker is not installed:

sudo apt update

sudo apt install docker.io -y

Start Docker:

sudo systemctl start docker

Enable Docker:

sudo systemctl enable docker

Check:

sudo systemctl status docker

Press:

q


19. TEST DOCKER
===============

Run:

sudo docker run hello-world

If Docker is working correctly, Docker will download the image and display the Hello World message.

This confirms that Docker is working.


20. GIVE JENKINS PERMISSION TO USE DOCKER
==========================================

This is an important step.

The Jenkins service normally runs using the Jenkins Linux user.

Docker access must be given to Jenkins.

Run:

sudo usermod -aG docker jenkins

Restart Jenkins:

sudo systemctl restart jenkins

Check Jenkins:

sudo systemctl status jenkins

Press:

q


21. VERIFY DOCKER ACCESS FOR JENKINS
====================================

Run:

sudo -u jenkins docker --version

If the command displays the Docker version, Jenkins can access Docker.

If you get a permission error:

Restart Jenkins again:

sudo systemctl restart jenkins

Then test:

sudo -u jenkins docker ps

If it displays the Docker container list without a permission error, continue.


22. CREATE THE PROJECT FOLDER ON YOUR COMPUTER
==============================================

On your local computer, create a folder:

jenkins-docker-project

Inside the folder create:

index.html
Dockerfile


23. CREATE index.html
=====================

Put the following content into index.html:

<!DOCTYPE html>
<html>
<head>
    <title>Jenkins Docker Project</title>
</head>
<body>
    <h1>Jenkins Deployment Successful!</h1>
    <p>This website was deployed automatically using Jenkins and Docker.</p>
    <p>GitHub → Jenkins → Docker → AWS EC2</p>
</body>
</html>


24. CREATE Dockerfile
=====================

Create a file named exactly:

Dockerfile

Do not add:

.txt

The file should contain:

FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80


Explanation:

FROM nginx:alpine

Uses a lightweight Nginx Docker image.

COPY

Copies our HTML file into the Nginx web directory.

EXPOSE 80

Documents that Nginx uses port 80 inside the container.


25. TEST THE DOCKER PROJECT LOCALLY
===================================

Open PowerShell or terminal inside the project folder.

Check files:

dir

You should see:

index.html
Dockerfile


26. BUILD DOCKER IMAGE
======================

Run:

docker build -t jenkins-web-app .

Explanation:

docker build
Builds a Docker image.

-t
Assigns a name/tag.

jenkins-web-app
Image name.

.
Uses the current directory as the build context.


27. CHECK THE IMAGE
===================

Run:

docker images

You should see:

jenkins-web-app


28. RUN THE CONTAINER LOCALLY
=============================

Run:

docker run -d --name jenkins-web-container -p 8081:80 jenkins-web-app

Explanation:

-d
Runs the container in the background.

--name
Gives the container a name.

-p 8081:80
Maps:

Computer port 8081
        ↓
Container port 80

jenkins-web-app
Docker image name.


29. OPEN THE WEBSITE LOCALLY
============================

Open:

http://localhost:8081

You should see:

Jenkins Deployment Successful!


30. CHECK THE CONTAINER
=======================

Run:

docker ps

You should see the running container.

Example:

jenkins-web-container


31. STOP THE TEST CONTAINER
===========================

Run:

docker stop jenkins-web-container

Then remove it:

docker rm jenkins-web-container

This is only the local test.

The Jenkins pipeline will perform similar operations automatically on the EC2 server.


32. CREATE GITHUB REPOSITORY
============================

Go to GitHub.

Create a new repository.

Example repository name:

jenkins-docker-project

Keep the repository public for this first beginner lab.

Do not add unnecessary files.

Create the repository.


33. CONNECT LOCAL PROJECT TO GITHUB
===================================

Open terminal inside:

jenkins-docker-project

Initialize Git:

git init

Check status:

git status

Add files:

git add .

Create the first commit:

git commit -m "Initial Jenkins Docker project"


34. SET MAIN BRANCH
===================

Run:

git branch -M main


35. ADD GITHUB REMOTE
=====================

Copy your GitHub repository URL.

Example:

https://github.com/YOUR-USERNAME/jenkins-docker-project.git

Run:

git remote add origin https://github.com/YOUR-USERNAME/jenkins-docker-project.git

Check:

git remote -v


36. PUSH CODE TO GITHUB
=======================

Run:

git push -u origin main

If GitHub asks for authentication, complete the GitHub authentication process.

Refresh the GitHub repository page.

You should see:

index.html
Dockerfile


37. IMPORTANT GITHUB CHECK
==========================

Before moving to Jenkins, make sure the GitHub repository contains:

index.html
Dockerfile

The Dockerfile must be named exactly:

Dockerfile

Not:

Dockerfile.txt


38. CREATE A JENKINS PIPELINE
=============================

Go back to Jenkins.

Open:

http://EC2-PUBLIC-IP:8080

From the Jenkins dashboard:

Click:

New Item


39. ENTER PROJECT NAME
======================

Enter:

jenkins-docker-project

Select:

Pipeline

Click:

OK


40. PIPELINE CONFIGURATION
==========================

Scroll down to:

Pipeline

Under:

Definition

Select:

Pipeline script


41. ENTER THE PIPELINE
======================

Use this pipeline:

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


42. CHANGE THE GITHUB URL
=========================

In the pipeline, find:

https://github.com/YOUR-USERNAME/jenkins-docker-project.git

Replace:

YOUR-USERNAME

with your actual GitHub username.

Example:

https://github.com/student123/jenkins-docker-project.git


43. SAVE THE PIPELINE
=====================

Click:

Save


44. RUN THE FIRST JENKINS BUILD
================================

Open the Jenkins project.

Click:

Build Now

Jenkins will start the pipeline.


45. WATCH THE BUILD
===================

You will see:

Build #1

Click:

Build #1

Then click:

Console Output

Watch the commands being executed.


46. UNDERSTAND THE PIPELINE STAGES
===================================

Stage 1:

Clone GitHub Repository

Jenkins downloads the code from GitHub.

Stage 2:

Build Docker Image

Jenkins runs:

docker build

Stage 3:

Stop Old Container

Jenkins tries to stop the previous container.

The command uses:

|| true

so the pipeline does not fail when the container does not exist during the first deployment.

Stage 4:

Run New Container

Jenkins runs:

docker run

The application starts inside Docker.


47. CHECK THE BUILD RESULT
==========================

At the end of Console Output, you should see:

Finished: SUCCESS

This means Jenkins completed all pipeline stages successfully.


48. CHECK DOCKER ON EC2
=======================

SSH into the EC2 server.

Run:

docker images

You should see:

jenkins-web-app


Run:

docker ps

You should see:

jenkins-web-container


49. OPEN THE DEPLOYED WEBSITE
=============================

Find your EC2 Public IPv4 address.

Open:

http://EC2-PUBLIC-IP:8081

Example:

http://13.XX.XX.XX:8081

You should see:

Jenkins Deployment Successful!


50. COMPLETE PROJECT FLOW
=========================

Your project is now:

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
Nginx Web Server
   |
   v
AWS EC2
   |
   v
http://EC2-PUBLIC-IP:8081


51. MAKE A CHANGE TO THE WEBSITE
=================================

Now test the real benefit of Jenkins.

On your local computer, edit:

index.html

Change:

Jenkins Deployment Successful!

to:

Jenkins Automatic Deployment Successful!


52. COMMIT THE CHANGE
=====================

Run:

git status

Then:

git add .

Commit:

git commit -m "Update website message"

Push:

git push


53. RUN JENKINS AGAIN
=====================

Go to Jenkins.

Open:

jenkins-docker-project

Click:

Build Now

Jenkins will:

1. Get the latest GitHub code.
2. Build a new Docker image.
3. Stop the old container.
4. Remove the old container.
5. Start the new container.


54. CHECK THE WEBSITE AGAIN
===========================

Open:

http://EC2-PUBLIC-IP:8081

Refresh the page.

The updated message should appear.


55. WHAT HAPPENED?
==================

Before Jenkins:

Developer
   ↓
GitHub
   ↓
Manual deployment
   ↓
Docker build
   ↓
Docker run

After Jenkins:

Developer
   ↓
GitHub
   ↓
Jenkins
   ↓
Docker build
   ↓
Docker run
   ↓
Application


56. WHY JENKINS IS USEFUL
=========================

Jenkins is useful for automation.

Without Jenkins, deployment steps may need to be performed manually.

With Jenkins, Jenkins can automatically execute predefined steps.

Examples:

- Get source code
- Run tests
- Build application
- Build Docker image
- Push Docker image
- Deploy application
- Restart services
- Run automated checks

Jenkins is commonly used as part of CI/CD.


57. CI AND CD BASIC IDEA
========================

CI = Continuous Integration

Developers frequently integrate code changes into a shared repository.

A CI process can automatically:

- Get the code
- Build the application
- Run tests
- Report failures


CD = Continuous Delivery / Continuous Deployment

The software is prepared or deployed through an automated process.

In this project:

GitHub
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Container
   ↓
EC2 Application

This is a simple CI/CD-style deployment pipeline.


58. IMPORTANT JENKINS TERMS
===========================

Jenkins:

Automation server.

Job:

A Jenkins task/project.

Pipeline:

A sequence of automated stages.

Stage:

A logical section of a pipeline.

Build:

One execution of a Jenkins job or pipeline.

Agent:

The machine where Jenkins executes pipeline steps.

Console Output:

The log showing what happened during a build.

Jenkinsfile:

A file that stores pipeline code.


59. WHY ARE WE USING "PIPELINE"?
================================

Jenkins supports different job types.

For DevOps projects, Pipeline is important because the deployment process can be written as code.

Example:

stage('Build') {
    steps {
        sh 'docker build -t myapp .'
    }
}

This makes the process repeatable.


60. BASIC PIPELINE STRUCTURE
============================

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


61. UNDERSTAND THE IMPORTANT KEYWORDS
=====================================

pipeline

Defines a Jenkins pipeline.

agent any

Allows Jenkins to execute the pipeline on an available agent.

stages

Contains the pipeline stages.

stage

Defines one logical step of the pipeline.

steps

Contains the commands Jenkins should execute.

sh

Runs a Linux shell command.


62. COMMON COMMANDS USED IN THIS LAB
=====================================

Git:

git init
git status
git add .
git commit -m "message"
git branch -M main
git remote add origin URL
git push -u origin main
git push


Docker:

docker build -t image-name .
docker images
docker ps
docker run -d --name container-name -p 8081:80 image-name
docker stop container-name
docker rm container-name


Jenkins:

Build Now
Console Output


63. TROUBLESHOOTING
===================

PROBLEM 1:
Jenkins page does not open.

Check:

sudo systemctl status jenkins

Check port:

sudo ss -lntp | grep 8080

Check AWS Security Group.

Make sure inbound TCP 8080 is allowed from your IP.


PROBLEM 2:
Application does not open.

Check:

docker ps

Check whether the container is running.

Then check:

docker logs jenkins-web-container


PROBLEM 3:
Port 8081 is already in use.

Check:

sudo ss -lntp | grep 8081

Also check:

docker ps

Stop/remove the container using the port if it is no longer required.


PROBLEM 4:
Jenkins cannot run Docker.

Test:

sudo -u jenkins docker ps

If permission is denied:

sudo usermod -aG docker jenkins

Then:

sudo systemctl restart jenkins

Test again.


PROBLEM 5:
GitHub clone fails.

Check:

- GitHub repository URL
- Branch name
- Repository visibility
- Branch is really named main
- Internet access from EC2

Test manually:

git clone https://github.com/YOUR-USERNAME/jenkins-docker-project.git


PROBLEM 6:
Docker build fails.

Check:

ls

You should have:

index.html
Dockerfile

Check Dockerfile spelling.

Run manually:

docker build -t jenkins-web-app .


PROBLEM 7:
Jenkins build fails at "Stop Old Container".

The pipeline already contains:

docker stop jenkins-web-container || true
docker rm jenkins-web-container || true

This allows the pipeline to continue when the old container does not exist.

If the failure occurs elsewhere, read the Console Output carefully.


64. SECURITY NOTES
==================

For a classroom beginner lab, the GitHub repository can be public.

For real projects:

- Do not store passwords in GitHub.
- Do not store AWS access keys in source code.
- Do not hard-code secrets in Jenkinsfiles.
- Use Jenkins Credentials for secrets.
- Restrict AWS Security Group access.
- Use HTTPS where appropriate.
- Use proper IAM permissions.
- Use separate environments such as development, staging and production.


65. OPTIONAL CLEANUP
====================

If you want to remove the Docker container:

docker stop jenkins-web-container

docker rm jenkins-web-container

Remove the image:

docker rmi jenkins-web-app

Do not remove Jenkins itself unless the lab is completely finished.


66. FINAL CHECKLIST
===================

AWS:

[ ] Ubuntu EC2 created
[ ] Security Group configured
[ ] Port 22 allowed
[ ] Port 8080 allowed
[ ] Port 8081 allowed

Linux:

[ ] Connected to EC2 using SSH
[ ] Ubuntu updated
[ ] Java installed

Jenkins:

[ ] Jenkins installed
[ ] Jenkins started
[ ] Jenkins enabled
[ ] Jenkins dashboard opened
[ ] Admin user created

Docker:

[ ] Docker installed
[ ] Docker service running
[ ] Jenkins can access Docker

Git/GitHub:

[ ] Git repository created
[ ] index.html created
[ ] Dockerfile created
[ ] Code pushed to GitHub

Jenkins:

[ ] Pipeline created
[ ] GitHub URL configured
[ ] Build executed
[ ] Console Output checked
[ ] Finished: SUCCESS

Deployment:

[ ] Docker image created
[ ] Docker container running
[ ] Application opened on port 8081
[ ] Website update tested through Jenkins


67. FINAL ARCHITECTURE
======================

                 +----------------+
                 |    Developer   |
                 +-------+--------+
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
                 |    Jenkins     |
                 |   Pipeline     |
                 +-------+--------+
                         |
                         | docker build
                         v
                 +----------------+
                 | Docker Image   |
                 +-------+--------+
                         |
                         | docker run
                         v
              +----------------------+
              |      AWS EC2         |
              |                      |
              |  Docker Container    |
              |       Nginx          |
              +----------+-----------+
                         |
                         | HTTP :8081
                         v
                 +----------------+
                 |  Web Browser   |
                 +----------------+


68. WHAT TO LEARN NEXT
======================

After completing this first project, the next Jenkins topics can be learned in this order:

1. Jenkins Freestyle Job
2. Jenkins Pipeline
3. Declarative Pipeline
4. Jenkinsfile
5. GitHub integration
6. Jenkins Credentials
7. Webhooks
8. Automatic build on git push
9. Build parameters
10. Environment variables
11. Docker image tagging
12. Docker Hub integration
13. Jenkins + AWS deployment
14. CI/CD pipeline
15. Jenkins + Docker + GitHub final project


END OF JENKINS LAB
============================

Main project flow:

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
Web Application

The main goal of this first project is to understand how Jenkins connects GitHub, Docker and AWS EC2 into one simple automated deployment workflow.
