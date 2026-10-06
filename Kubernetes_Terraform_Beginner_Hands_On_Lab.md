# Kubernetes + Terraform Beginner Hands-On Lab
## Simple DevOps Project — Deploy a Web App with Kubernetes and Provision AWS with Terraform

---

# 0. COURSE FLOW

This lab is designed as the next step after:

```text
GitHub
   ↓
Jenkins
   ↓
Docker
   ↓
AWS EC2
   ↓
Web Application
```

Now we will learn:

```text
                    TERRAFORM
                       |
                       | Create AWS infrastructure
                       v
                    AWS EC2
                       |
                       | Install Kubernetes
                       v
                  KUBERNETES
                       |
              +--------+--------+
              |                 |
           Deployment         Service
              |                 |
              v                 v
             Pod          Browser Access
              |
              v
        Nginx Web App
```

## What students will learn

### Kubernetes
- What Kubernetes is
- Cluster, Node, Pod
- Deployment
- Service
- Namespace
- `kubectl`
- YAML files
- Scaling
- Updating an application
- Rolling update
- Rollback
- Logs
- Troubleshooting
- Cleanup

### Terraform
- What Infrastructure as Code means
- Terraform workflow
- Provider
- Resource
- Variables
- Outputs
- `terraform init`
- `terraform plan`
- `terraform apply`
- `terraform destroy`
- Creating an AWS EC2 instance
- Understanding Terraform state

---

# 1. IMPORTANT: WHAT WE ARE BUILDING

We will create a very simple web application.

The application is:

```text
Nginx
  |
  v
HTML Page
  |
  v
Kubernetes Pod
  |
  v
Kubernetes Service
  |
  v
Browser
```

The page will display:

```text
DevOps Kubernetes Project
Version 1
```

Later we will change it to:

```text
DevOps Kubernetes Project
Version 2
```

Then we will demonstrate rollback.

---

# 2. SIMPLE THEORY — WHAT IS KUBERNETES?

Kubernetes is a platform used to manage containers.

Docker can run a container:

```text
docker run nginx
```

But in production we may have:

```text
100 containers
10 servers
Multiple applications
Scaling requirements
Application failures
Updates
Rollback
Networking
```

Managing all of these manually becomes difficult.

Kubernetes helps manage containers automatically.

Simple definition:

> Kubernetes is a container orchestration platform.

---

# 3. DOCKER VS KUBERNETES

## Docker

Docker mainly helps us:

```text
Build Image
    ↓
Run Container
    ↓
Manage Container
```

## Kubernetes

Kubernetes helps us:

```text
Manage Containers
        ↓
Deploy
Scale
Restart
Network
Update
Rollback
```

Think:

```text
Docker       = Container technology

Kubernetes   = Container orchestration
```

---

# 4. IMPORTANT KUBERNETES TERMS

## 4.1 Cluster

A Kubernetes cluster is the complete Kubernetes environment.

```text
Kubernetes Cluster
        |
        +--- Node
        |
        +--- Node
        |
        +--- Node
```

For our beginner lab, we will use one node.

---

## 4.2 Node

A node is a machine that runs Kubernetes workloads.

Example:

```text
AWS EC2
   |
   └── Kubernetes Node
```

---

## 4.3 Pod

A Pod is the smallest deployable unit in Kubernetes.

For our lab:

```text
Pod
 |
 └── Nginx Container
```

Remember:

```text
Kubernetes
   ↓
Pod
   ↓
Container
```

---

## 4.4 Deployment

A Deployment manages Pods.

Example:

```text
Deployment
    |
    +--- Pod
    |
    +--- Pod
    |
    +--- Pod
```

If we request 3 replicas:

```text
replicas: 3
```

Kubernetes tries to maintain 3 Pods.

---

## 4.5 Service

A Pod IP can change.

Therefore users should not directly depend on a Pod IP.

A Service provides stable network access.

```text
Browser
   |
   v
Service
   |
   +--- Pod
   |
   +--- Pod
   |
   +--- Pod
```

---

# 5. WHAT IS kubectl?

`kubectl` is the command-line tool used to communicate with Kubernetes.

Examples:

```bash
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get services
```

Think:

```text
You
 |
 | kubectl
 v
Kubernetes Cluster
```

---

# 6. LAB ARCHITECTURE

We will use:

```text
AWS EC2 Ubuntu
      |
      v
     k3s
      |
      v
Kubernetes
      |
      +--- Deployment
      |       |
      |       +--- Pod
      |              |
      |              +--- Nginx
      |
      +--- Service
              |
              v
           Browser
```

## Why k3s?

For teaching, k3s is a lightweight Kubernetes distribution.

It is much simpler for a beginner lab than building a full multi-node Kubernetes cluster.

---

# 7. REQUIREMENTS

Students need:

- AWS account
- One Ubuntu EC2 instance
- SSH access
- Internet connection
- Basic Linux commands
- Basic Docker knowledge

We will install the Kubernetes tools on the Ubuntu EC2 machine.

No local Kubernetes installation is required.

---

# 8. CREATE UBUNTU EC2

Use an Ubuntu Server EC2 instance.

Recommended beginner lab:

```text
OS:
Ubuntu Server 24.04 LTS

Architecture:
64-bit x86

Instance:
t3.small or equivalent

Storage:
20 GB
```

For a very small lab, students may use another suitable instance size depending on AWS availability and account limits.

---

# 9. AWS SECURITY GROUP

For the Kubernetes demo, allow:

```text
SSH
TCP
22
Your IP
```

For browser access to the web application:

```text
TCP
30080
0.0.0.0/0
```

Do NOT open unnecessary ports.

---

# 10. CONNECT TO EC2

From your terminal:

```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Example:

```bash
ssh -i devops-key.pem ubuntu@13.233.100.10
```

---

# 11. CHECK THE SERVER

Run:

```bash
whoami
```

Expected:

```text
ubuntu
```

Check OS:

```bash
cat /etc/os-release
```

Check CPU:

```bash
nproc
```

Check memory:

```bash
free -h
```

---

# 12. UPDATE UBUNTU

Run:

```bash
sudo apt update
```

Then:

```bash
sudo apt upgrade -y
```

---

# 13. INSTALL BASIC TOOLS

Run:

```bash
sudo apt install -y curl wget git
```

Check:

```bash
curl --version
git --version
```

---

# 14. INSTALL K3S

We will use k3s for the Kubernetes lab.

Run:

```bash
curl -sfL https://get.k3s.io | sh -
```

Wait for installation to finish.

---

# 15. CHECK K3S SERVICE

Run:

```bash
sudo systemctl status k3s
```

You should see:

```text
active (running)
```

Press:

```text
q
```

to exit the status screen.

---

# 16. CHECK KUBERNETES VERSION

Run:

```bash
sudo k3s kubectl version
```

---

# 17. MAKE kubectl EASIER

Create a kubectl shortcut:

```bash
sudo ln -s /usr/local/bin/k3s /usr/local/bin/kubectl
```

If the link already exists, do not recreate it.

Check:

```bash
kubectl version
```

---

# 18. CHECK KUBERNETES NODE

Run:

```bash
kubectl get nodes
```

Expected concept:

```text
NAME        STATUS   ROLES
ip-xxx      Ready    control-plane
```

The important value is:

```text
STATUS = Ready
```

---

# 19. FIRST KUBERNETES COMMANDS

Run:

```bash
kubectl get pods
```

Run:

```bash
kubectl get all
```

Run:

```bash
kubectl get namespaces
```

Explain to students:

```text
kubectl
   |
   +--- get nodes
   +--- get pods
   +--- get services
   +--- get deployments
   +--- get namespaces
```

---

# 20. CREATE PROJECT DIRECTORY

Run:

```bash
mkdir kubernetes-terraform-demo
```

Go inside:

```bash
cd kubernetes-terraform-demo
```

Create folders:

```bash
mkdir k8s
mkdir terraform
```

Final structure:

```text
kubernetes-terraform-demo/
|
+--- k8s/
|
+--- terraform/
|
+--- README.md
```

---

# 21. CREATE NAMESPACE

Create:

```text
k8s/namespace.yaml
```

Content:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-demo
```

---

# 22. UNDERSTAND THE NAMESPACE YAML

```yaml
apiVersion: v1
```

Tells Kubernetes which API version to use.

```yaml
kind: Namespace
```

Tells Kubernetes what object we are creating.

```yaml
metadata:
```

Contains object information.

```yaml
name: devops-demo
```

Namespace name.

---

# 23. CREATE DEPLOYMENT

Create:

```text
k8s/deployment.yaml
```

Use:

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
          image: nginx:alpine

          ports:
            - containerPort: 80
```

---

# 24. UNDERSTAND DEPLOYMENT YAML

Important:

```yaml
kind: Deployment
```

We are creating a Deployment.

```yaml
replicas: 2
```

We want two Pods.

```yaml
image: nginx:alpine
```

The Pod will run an Nginx container.

```yaml
containerPort: 80
```

Nginx listens on port 80 inside the container.

---

# 25. CREATE SERVICE

Create:

```text
k8s/service.yaml
```

Use:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: webapp-service
  namespace: devops-demo

spec:
  type: NodePort

  selector:
    app: webapp

  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

---

# 26. UNDERSTAND SERVICE YAML

The Service connects users to Pods.

```text
Browser
   |
   v
NodePort 30080
   |
   v
Service
   |
   v
Pod
   |
   v
Nginx
```

Important:

```yaml
type: NodePort
```

This exposes the application through a port on the Kubernetes node.

```yaml
nodePort: 30080
```

We will access:

```text
http://EC2_PUBLIC_IP:30080
```

---

# 27. DEPLOY NAMESPACE

Run:

```bash
kubectl apply -f k8s/namespace.yaml
```

Expected:

```text
namespace/devops-demo created
```

Check:

```bash
kubectl get namespaces
```

---

# 28. DEPLOY APPLICATION

Run:

```bash
kubectl apply -f k8s/deployment.yaml
```

Expected:

```text
deployment.apps/webapp created
```

---

# 29. CREATE SERVICE

Run:

```bash
kubectl apply -f k8s/service.yaml
```

Expected:

```text
service/webapp-service created
```

---

# 30. CHECK DEPLOYMENT

Run:

```bash
kubectl get deployments -n devops-demo
```

Expected:

```text
NAME      READY
webapp    2/2
```

---

# 31. CHECK PODS

Run:

```bash
kubectl get pods -n devops-demo
```

Expected:

```text
webapp-xxxxx   Running
webapp-yyyyy   Running
```

The exact Pod names will be different.

---

# 32. CHECK SERVICE

Run:

```bash
kubectl get services -n devops-demo
```

Expected concept:

```text
NAME             TYPE       PORT
webapp-service   NodePort   80:30080
```

---

# 33. OPEN THE APPLICATION

Find the EC2 public IP.

Example:

```text
13.233.100.10
```

Open:

```text
http://13.233.100.10:30080
```

You should see the default Nginx page.

---

# 34. IMPORTANT OBSERVATION

We did NOT run:

```bash
docker run nginx
```

Instead:

```text
Kubernetes
    ↓
Deployment
    ↓
Pod
    ↓
Nginx Container
```

This is the key concept.

---

# 35. CHECK POD DETAILS

Run:

```bash
kubectl describe pod -n devops-demo
```

This is useful when troubleshooting.

For a specific Pod:

```bash
kubectl get pods -n devops-demo
```

Then:

```bash
kubectl describe pod POD_NAME -n devops-demo
```

---

# 36. VIEW LOGS

First get Pods:

```bash
kubectl get pods -n devops-demo
```

Then:

```bash
kubectl logs POD_NAME -n devops-demo
```

---

# 37. SCALE THE APPLICATION

Currently:

```yaml
replicas: 2
```

Change it to:

```yaml
replicas: 3
```

Apply again:

```bash
kubectl apply -f k8s/deployment.yaml
```

Check:

```bash
kubectl get pods -n devops-demo
```

Expected:

```text
3 Pods
```

---

# 38. SCALE USING kubectl

We can also scale directly:

```bash
kubectl scale deployment webapp --replicas=4 -n devops-demo
```

Check:

```bash
kubectl get pods -n devops-demo
```

You should see four Pods.

---

# 39. SCALE BACK

Run:

```bash
kubectl scale deployment webapp --replicas=2 -n devops-demo
```

---

# 40. CREATE VERSION 2

For a better application demo, create:

```text
k8s/configmap.yaml
```

Use:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: web-content
  namespace: devops-demo

data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>DevOps Kubernetes Project</title>
    </head>
    <body>
      <h1>DevOps Kubernetes Project</h1>
      <h2>Version 1</h2>
      <p>Application running on Kubernetes.</p>
    </body>
    </html>
```

---

# 41. APPLY CONFIGMAP

Run:

```bash
kubectl apply -f k8s/configmap.yaml
```

---

# 42. UPDATE DEPLOYMENT

Replace `k8s/deployment.yaml` with:

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
          image: nginx:alpine

          ports:
            - containerPort: 80

          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html

      volumes:
        - name: web-content
          configMap:
            name: web-content
```

Apply:

```bash
kubectl apply -f k8s/deployment.yaml
```

---

# 43. CHECK ROLLOUT

Run:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

Expected:

```text
deployment "webapp" successfully rolled out
```

Open:

```text
http://EC2_PUBLIC_IP:30080
```

Expected:

```text
DevOps Kubernetes Project
Version 1
Application running on Kubernetes.
```

---

# 44. VERSION 2

Edit:

```text
k8s/configmap.yaml
```

Change:

```text
Version 1
```

to:

```text
Version 2
```

Apply:

```bash
kubectl apply -f k8s/configmap.yaml
```

Restart the Deployment so the Pods load the updated ConfigMap:

```bash
kubectl rollout restart deployment/webapp -n devops-demo
```

Check:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

Refresh the browser.

---

# 45. ROLLOUT HISTORY

Run:

```bash
kubectl rollout history deployment/webapp -n devops-demo
```

This shows Deployment revision history.

---

# 46. ROLLBACK

If Version 2 has a problem:

```bash
kubectl rollout undo deployment/webapp -n devops-demo
```

Check:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

Then:

```bash
kubectl rollout history deployment/webapp -n devops-demo
```

IMPORTANT:

For this ConfigMap demonstration, rollback of the Deployment alone does not automatically restore the ConfigMap contents. This is an important teaching point: Kubernetes object versioning and application configuration versioning must be managed together.

---

# 47. KUBERNETES TROUBLESHOOTING COMMANDS

Teach these commands:

## Nodes

```bash
kubectl get nodes
```

## Pods

```bash
kubectl get pods -n devops-demo
```

## Deployments

```bash
kubectl get deployments -n devops-demo
```

## Services

```bash
kubectl get services -n devops-demo
```

## Detailed Pod information

```bash
kubectl describe pod POD_NAME -n devops-demo
```

## Logs

```bash
kubectl logs POD_NAME -n devops-demo
```

## All resources

```bash
kubectl get all -n devops-demo
```

## Events

```bash
kubectl get events -n devops-demo
```

---

# 48. COMMON KUBERNETES PROBLEMS

## Problem 1 — Pod is Pending

Check:

```bash
kubectl describe pod POD_NAME -n devops-demo
```

Look at the Events section.

---

## Problem 2 — Pod is CrashLoopBackOff

Check:

```bash
kubectl logs POD_NAME -n devops-demo
```

---

## Problem 3 — Service does not work

Check:

```bash
kubectl get services -n devops-demo
```

Then:

```bash
kubectl get pods -n devops-demo --show-labels
```

The Service selector must match the Pod label:

```yaml
selector:
  app: webapp
```

and:

```yaml
labels:
  app: webapp
```

---

# 49. KUBERNETES THEORY RECAP

Teach students this flow:

```text
kubectl apply
      ↓
Deployment
      ↓
ReplicaSet
      ↓
Pod
      ↓
Container
```

Networking:

```text
Browser
   ↓
NodePort
   ↓
Service
   ↓
Pod
   ↓
Container
```

---

# 50. NOW START TERRAFORM

Kubernetes manages containers.

Terraform manages infrastructure.

Simple difference:

```text
Terraform
   ↓
Infrastructure

Kubernetes
   ↓
Containers / Applications
```

Example:

```text
Terraform
   ↓
AWS EC2

Kubernetes
   ↓
Pod
   ↓
Application
```

---

# 51. WHAT IS INFRASTRUCTURE AS CODE?

Normally, we may create infrastructure manually:

```text
AWS Console
   ↓
Launch EC2
   ↓
Select Ubuntu
   ↓
Select Instance Type
   ↓
Configure Network
   ↓
Create
```

With Terraform:

```text
Terraform Code
      ↓
terraform apply
      ↓
AWS Infrastructure
```

This is called:

> Infrastructure as Code (IaC)

---

# 52. TERRAFORM WORKFLOW

The most important Terraform flow is:

```text
Write Code
   ↓
terraform init
   ↓
terraform plan
   ↓
terraform apply
   ↓
Infrastructure Created
```

When finished:

```text
terraform destroy
```

---

# 53. TERRAFORM TERMS

## Provider

Tells Terraform which platform we are managing.

Example:

```text
AWS
```

---

## Resource

An infrastructure object.

Example:

```text
aws_instance
```

---

## Variable

Allows reusable input.

Example:

```text
instance_type
```

---

## Output

Displays useful information after deployment.

Example:

```text
public_ip
```

---

## State

Terraform keeps track of infrastructure using state.

Important file:

```text
terraform.tfstate
```

Do not casually delete or manually edit this file.

---

# 54. INSTALL TERRAFORM ON UBUNTU

On the EC2 machine:

```bash
sudo apt update
```

Install prerequisites:

```bash
sudo apt install -y gnupg software-properties-common curl
```

Add the official HashiCorp repository:

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

Add repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
```

Update:

```bash
sudo apt update
```

Install Terraform:

```bash
sudo apt install -y terraform
```

Check:

```bash
terraform version
```

---

# 55. IMPORTANT AWS CREDENTIAL CONCEPT

Terraform needs permission to create AWS resources.

For a classroom environment, students should use an AWS identity with only the permissions required for the lab.

Do NOT put AWS access keys directly inside Terraform code.

Never write:

```text
access_key = "MY_SECRET_KEY"
secret_key = "MY_SECRET"
```

inside a GitHub repository.

---

# 56. SIMPLE TERRAFORM PROJECT

Go to:

```bash
cd ~/kubernetes-terraform-demo/terraform
```

Create:

```text
main.tf
```

---

# 57. TERRAFORM PROVIDER

Put this in `main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

---

# 58. CREATE VARIABLES

Create:

```text
variables.tf
```

Use:

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

---

# 59. CREATE EC2 RESOURCE

Add to `main.tf`:

```hcl
resource "aws_instance" "devops_server" {
  ami           = "ami-0f58b397bc5c1f2e8"
  instance_type = var.instance_type

  tags = {
    Name = "terraform-devops-server"
  }
}
```

IMPORTANT:

The AMI ID is region-specific.

The example above is for demonstration and may change.

Students MUST select a valid Ubuntu AMI ID for their selected AWS region.

Do not blindly reuse an old AMI ID.

---

# 60. CREATE OUTPUT

Create:

```text
outputs.tf
```

Use:

```hcl
output "instance_id" {
  value = aws_instance.devops_server.id
}

output "public_ip" {
  value = aws_instance.devops_server.public_ip
}
```

---

# 61. TERRAFORM PROJECT STRUCTURE

Final structure:

```text
terraform/
|
+--- main.tf
+--- variables.tf
+--- outputs.tf
```

---

# 62. TERRAFORM INIT

Inside the Terraform directory:

```bash
terraform init
```

Terraform downloads the AWS provider.

Expected concept:

```text
Terraform
   ↓
AWS Provider
   ↓
Ready
```

---

# 63. TERRAFORM FORMAT

Run:

```bash
terraform fmt
```

This formats Terraform files.

---

# 64. TERRAFORM VALIDATE

Run:

```bash
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

---

# 65. TERRAFORM PLAN

Run:

```bash
terraform plan
```

Terraform shows what it intends to create.

Example:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

IMPORTANT:

`plan` does NOT create the infrastructure.

---

# 66. TERRAFORM APPLY

Run:

```bash
terraform apply
```

Terraform will ask for confirmation.

Type:

```text
yes
```

Terraform creates the EC2 instance.

---

# 67. CHECK TERRAFORM OUTPUT

Run:

```bash
terraform output
```

You should see values such as:

```text
instance_id
public_ip
```

---

# 68. CHECK TERRAFORM STATE

Run:

```bash
ls
```

You will see:

```text
main.tf
variables.tf
outputs.tf
terraform.tfstate
```

Explain:

```text
terraform.tfstate
```

is Terraform's record of the infrastructure it manages.

---

# 69. TERRAFORM PLAN AGAIN

Run:

```bash
terraform plan
```

If nothing changed, Terraform should show that there are no changes to make.

This demonstrates:

```text
Terraform Code
       +
Terraform State
       ↓
Compare with AWS
       ↓
Required Changes
```

---

# 70. CHANGE INFRASTRUCTURE

Change:

```hcl
instance_type = "t3.micro"
```

to:

```hcl
instance_type = "t3.small"
```

Run:

```bash
terraform plan
```

Terraform will show the change it plans to make.

Then:

```bash
terraform apply
```

Confirm:

```text
yes
```

---

# 71. TERRAFORM DESTROY

When the lab is finished:

```bash
terraform destroy
```

Confirm:

```text
yes
```

This removes infrastructure managed by Terraform.

IMPORTANT:

Only run this when you are certain the resource is the lab resource.

---

# 72. TERRAFORM SAFETY RULES

Never commit:

```text
terraform.tfstate
terraform.tfstate.*
.terraform/
*.tfvars
```

if those files contain secrets or sensitive environment-specific values.

Create:

```text
.gitignore
```

Use:

```gitignore
.terraform/
terraform.tfstate
terraform.tfstate.*
*.tfvars
crash.log
```

---

# 73. IMPORTANT NOTE ABOUT THIS BEGINNER LAB

For the first Terraform class, do NOT try to make Terraform install and configure the complete Kubernetes cluster automatically.

That introduces too many concepts at once.

Teach in this order:

```text
Class 1
Linux + Kubernetes installation
        ↓
kubectl
        ↓
Pod
        ↓
Deployment
        ↓
Service
        ↓
Scaling
        ↓
Rollback
```

Then:

```text
Class 2
Terraform
        ↓
Provider
        ↓
Resource
        ↓
Variables
        ↓
Outputs
        ↓
Init
        ↓
Plan
        ↓
Apply
        ↓
Destroy
```

Later:

```text
Class 3+
Terraform + AWS + Kubernetes
```

---

# 74. COMPLETE STUDENT PRACTICAL

Students must complete:

## Kubernetes

- [ ] Create Ubuntu EC2
- [ ] Install k3s
- [ ] Check Kubernetes node
- [ ] Create namespace
- [ ] Create Deployment
- [ ] Create Service
- [ ] Deploy Nginx
- [ ] Access application from browser
- [ ] Check Pods
- [ ] Check logs
- [ ] Scale from 2 to 4 Pods
- [ ] Scale back to 2
- [ ] Perform application update
- [ ] Check rollout
- [ ] Demonstrate rollback concept
- [ ] Troubleshoot a Pod

## Terraform

- [ ] Install Terraform
- [ ] Create provider
- [ ] Create variables
- [ ] Create AWS EC2 resource
- [ ] Create outputs
- [ ] Run `terraform init`
- [ ] Run `terraform fmt`
- [ ] Run `terraform validate`
- [ ] Run `terraform plan`
- [ ] Run `terraform apply`
- [ ] Check output
- [ ] Run `terraform plan` again
- [ ] Change instance type
- [ ] Apply the change
- [ ] Run `terraform destroy`

---

# 75. COMMAND CHEAT SHEET

## Kubernetes

```bash
kubectl get nodes

kubectl get pods -n devops-demo

kubectl get deployments -n devops-demo

kubectl get services -n devops-demo

kubectl get all -n devops-demo

kubectl describe pod POD_NAME -n devops-demo

kubectl logs POD_NAME -n devops-demo

kubectl scale deployment webapp --replicas=4 -n devops-demo

kubectl rollout status deployment/webapp -n devops-demo

kubectl rollout history deployment/webapp -n devops-demo

kubectl rollout undo deployment/webapp -n devops-demo
```

## Terraform

```bash
terraform init

terraform fmt

terraform validate

terraform plan

terraform apply

terraform output

terraform show

terraform destroy
```

---

# 76. FINAL ARCHITECTURE

```text
                         DEVELOPER
                             |
                             | Code
                             v
                         GitHub
                             |
                             |
                      DevOps Engineer
                             |
              +--------------+--------------+
              |                             |
              v                             v
         Terraform                      kubectl
              |                             |
              v                             v
             AWS                         k3s
              |                             |
              v                             v
             EC2                     Kubernetes Cluster
                                            |
                               +------------+------------+
                               |                         |
                               v                         v
                          Deployment                  Service
                               |                         |
                               v                         |
                             Pods <----------------------+
                               |
                               v
                         Nginx Container
                               |
                               v
                           Web Browser
```

---

# 77. TEACHING FLOW FOR YOU

Use this order in the classroom.

## Part A — Kubernetes Theory

Explain:

```text
Container
   ↓
Pod
   ↓
Deployment
   ↓
Service
   ↓
Cluster
```

Do not start with YAML immediately.

---

## Part B — Kubernetes Hands-On

Run:

```text
EC2
 ↓
Install k3s
 ↓
kubectl get nodes
 ↓
Namespace
 ↓
Deployment
 ↓
Pod
 ↓
Service
 ↓
Browser
```

---

## Part C — Kubernetes Operations

Teach:

```text
get
describe
logs
scale
rollout
rollback
```

---

## Part D — Terraform Theory

Explain:

```text
Manual Infrastructure
        ↓
Infrastructure as Code
        ↓
Terraform
```

Then explain:

```text
Provider
Resource
Variable
Output
State
```

---

## Part E — Terraform Hands-On

Run:

```text
terraform init
       ↓
terraform fmt
       ↓
terraform validate
       ↓
terraform plan
       ↓
terraform apply
       ↓
terraform output
       ↓
terraform plan
       ↓
terraform destroy
```

---

# 78. INTERVIEW QUESTIONS

## Kubernetes

### Q1. What is Kubernetes?

Kubernetes is a container orchestration platform.

### Q2. What is a Pod?

A Pod is the smallest deployable unit in Kubernetes.

### Q3. What is a Deployment?

A Deployment manages the desired number of Pods and supports application updates.

### Q4. Why do we need a Service?

A Service provides stable networking to Pods.

### Q5. What is kubectl?

`kubectl` is the command-line tool used to communicate with Kubernetes.

### Q6. What is scaling?

Increasing or decreasing the number of application Pods.

---

## Terraform

### Q7. What is Terraform?

Terraform is an Infrastructure as Code tool.

### Q8. What is a provider?

A provider allows Terraform to interact with a platform such as AWS.

### Q9. What is a resource?

A resource represents infrastructure managed by Terraform.

### Q10. What does terraform plan do?

It shows the changes Terraform intends to make.

### Q11. What does terraform apply do?

It applies the planned infrastructure changes.

### Q12. What does terraform destroy do?

It removes infrastructure managed by the Terraform configuration.

### Q13. What is terraform.tfstate?

It stores Terraform's state information about managed infrastructure.

---

# 79. FINAL STUDENT CHALLENGE

Create your own project:

```text
Project Name:
student-kubernetes-app
```

Requirements:

```text
1. Create a namespace
2. Create a Deployment
3. Run 3 replicas
4. Create a NodePort Service
5. Deploy an Nginx application
6. Access it from browser
7. Scale to 5 Pods
8. Scale back to 2
9. Update the application
10. Check rollout status
11. Demonstrate rollback
```

Terraform challenge:

```text
1. Create AWS provider
2. Create EC2 resource
3. Use variables
4. Create outputs
5. Run init
6. Run plan
7. Run apply
8. Verify AWS resource
9. Change instance type
10. Run plan
11. Apply change
12. Destroy resource
```

---

# 80. MOST IMPORTANT CONCEPT TO REMEMBER

```text
                    DEVOPS
                       |
          +------------+------------+
          |                         |
          v                         v
      TERRAFORM                 KUBERNETES
          |                         |
          v                         v
   Infrastructure              Applications
          |                         |
          v                         v
        AWS EC2                    Pods
                                   |
                                   v
                               Containers
```

Terraform answers:

> "How do I create and manage infrastructure?"

Kubernetes answers:

> "How do I run and manage containerized applications?"

Together:

```text
Terraform
    ↓
AWS Infrastructure
    ↓
Kubernetes
    ↓
Containers
    ↓
Application
```

# END OF BEGINNER LAB
