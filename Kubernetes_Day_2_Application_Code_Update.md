# Kubernetes Day 2 — Update Application Code on the Same EC2 Server

## 3-Hour Hands-On Project

**Project:** Kubernetes Application Code Update, Rolling Update, Verification & Rollback  
**Environment:** Existing AWS EC2 Ubuntu server + k3s + kubectl  
**Previous lab:** Kubernetes + Terraform Beginner Hands-On Lab  
**Important:** Do **not** create a new EC2 instance for this lab.

---

# 0. What We Did in the Previous Lab

In the previous lab, the application was running on the same EC2 server using:

```text
AWS EC2
   |
   v
k3s Kubernetes
   |
   v
Deployment
   |
   v
Nginx Pod
   |
   v
Service
   |
   v
Browser
```

The previous project used:

```text
Namespace:
devops-demo

Deployment:
webapp

Service:
webapp-service

Service Port:
30080

Application:
Nginx
```

The application was updated from:

```text
Version 1
```

to:

```text
Version 2
```

using a ConfigMap.

This lab continues from that exact environment.

---

# 1. Today's Goal

Today we will learn how a DevOps engineer changes application code **without creating a new server**.

We will create a small web application containing:

```text
index.html
style.css
script.js
```

Then we will:

1. Check the existing Kubernetes application.
2. Understand where the application code is stored.
3. Create Version 3 of the application.
4. Store the application files in a Kubernetes ConfigMap.
5. Update the existing Deployment.
6. Restart the application Pods safely.
7. Verify the new application.
8. Watch the rollout.
9. Make another application change.
10. Intentionally create a bad application change.
11. Troubleshoot it.
12. Restore the working application.
13. Scale the application.
14. Verify that all replicas serve the same application.
15. Understand the difference between application code, container image, ConfigMap, Deployment and Service.
16. Finish with a mini DevOps challenge.

---

# 2. Important Concept

We are **not** doing this:

```text
New Code
   |
   v
New EC2
   |
   v
Install Kubernetes Again
```

Instead:

```text
Same EC2
   |
   v
Same Kubernetes Cluster
   |
   v
Same Deployment
   |
   v
Same Service
   |
   v
Change Application
   |
   v
Update Pods
```

This is an important real-world DevOps concept.

---

# 3. 3-HOUR CLASS FLOW

## Hour 1 — Understand + Inspect Existing Application

### 00:00 – 00:15
Review previous architecture.

### 00:15 – 00:30
Connect to existing EC2 and check Kubernetes.

### 00:30 – 00:45
Inspect Deployment, Service and Pods.

### 00:45 – 01:00
Understand where application code is stored.

---

## Hour 2 — Change Application Code

### 01:00 – 01:15
Create HTML/CSS/JavaScript application.

### 01:15 – 01:30
Create/update ConfigMap.

### 01:30 – 01:45
Update Deployment.

### 01:45 – 02:00
Roll out the new application and verify it.

---

## Hour 3 — DevOps Operations

### 02:00 – 02:15
Change application to Version 4.

### 02:15 – 02:30
Observe rollout and Pods.

### 02:30 – 02:45
Create a controlled problem and troubleshoot.

### 02:45 – 03:00
Restore application, scale Pods and complete challenge.

---

# 4. STEP 1 — CONNECT TO THE SAME EC2 SERVER

From your local computer:

```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Example:

```bash
ssh -i devops-key.pem ubuntu@13.233.100.10
```

Use the **same EC2 server from the previous class**.

Do not launch another EC2 instance.

---

# 5. STEP 2 — CHECK THE SERVER

Run:

```bash
whoami
```

Expected:

```text
ubuntu
```

Check the server:

```bash
hostname
```

Check the OS:

```bash
cat /etc/os-release
```

---

# 6. STEP 3 — CHECK KUBERNETES

Run:

```bash
kubectl get nodes
```

Expected:

```text
NAME        STATUS   ROLES
ip-xxx      Ready    control-plane
```

The important value is:

```text
Ready
```

If the node is not Ready, stop here and troubleshoot before continuing.

---

# 7. STEP 4 — CHECK THE EXISTING APPLICATION

Run:

```bash
kubectl get all -n devops-demo
```

You should see resources such as:

```text
pod
service
deployment
replicaset
```

Now check Pods:

```bash
kubectl get pods -n devops-demo
```

Expected concept:

```text
webapp-xxxxx   Running
webapp-yyyyy   Running
```

---

# 8. STEP 5 — CHECK THE DEPLOYMENT

Run:

```bash
kubectl get deployment webapp -n devops-demo
```

Expected:

```text
NAME     READY   UP-TO-DATE   AVAILABLE
webapp   2/2
```

Now inspect the Deployment:

```bash
kubectl describe deployment webapp -n devops-demo
```

Look for:

```text
Replicas
Pod Template
Containers
Volumes
```

---

# 9. STEP 6 — CHECK THE SERVICE

Run:

```bash
kubectl get service webapp-service -n devops-demo
```

Expected concept:

```text
webapp-service   NodePort   80:30080
```

The application is accessed using:

```text
http://EC2_PUBLIC_IP:30080
```

Example:

```text
http://13.233.100.10:30080
```

Open the URL in the browser.

---

# 10. STEP 7 — UNDERSTAND THE CURRENT FLOW

The current application flow is:

```text
Browser
   |
   | HTTP :30080
   v
Kubernetes Service
   |
   v
Deployment
   |
   v
Pod
   |
   v
Nginx
   |
   v
HTML
```

Today we will change:

```text
HTML
CSS
JavaScript
```

without changing:

```text
EC2
k3s
Namespace
Service
```

---

# 11. STEP 8 — GO TO THE PROJECT DIRECTORY

Run:

```bash
cd ~/kubernetes-terraform-demo
```

Check:

```bash
pwd
```

Expected:

```text
/home/ubuntu/kubernetes-terraform-demo
```

Check files:

```bash
ls
```

Expected concept:

```text
k8s
terraform
```

Go to Kubernetes files:

```bash
cd k8s
```

Check:

```bash
ls
```

You may see:

```text
namespace.yaml
deployment.yaml
service.yaml
configmap.yaml
```

---

# 12. STEP 9 — BACK UP THE CURRENT CONFIGURATION

Before changing anything, create a backup directory:

```bash
mkdir -p backup-version
```

Copy the current files:

```bash
cp deployment.yaml backup-version/deployment.yaml
```

If the ConfigMap exists:

```bash
cp configmap.yaml backup-version/configmap.yaml
```

This is a good DevOps practice.

---

# 13. STEP 10 — CREATE THE NEW APPLICATION

We will create a simple professional web page.

The application will contain:

```text
index.html
style.css
script.js
```

For Kubernetes, we will store these files inside a ConfigMap.

---

# 14. STEP 11 — CREATE VERSION 3 APPLICATION

Create/edit:

```text
configmap.yaml
```

Use the following complete content:

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
      <title>DevOps Kubernetes Application</title>
      <link rel="stylesheet" href="style.css">
    </head>

    <body>

      <div class="container">

        <h1>DevOps Kubernetes Project</h1>

        <h2>Version 3</h2>

        <p>
          Application code has been updated on the existing EC2 server.
        </p>

        <button onclick="showMessage()">
          Test Application
        </button>

        <p id="message"></p>

      </div>

      <script src="script.js"></script>

    </body>
    </html>

  style.css: |
    body {
      font-family: Arial, sans-serif;
      margin: 0;
      padding: 0;
      background: #f4f7fb;
    }

    .container {
      width: 70%;
      margin: 100px auto;
      padding: 40px;
      text-align: center;
      background: white;
      border-radius: 12px;
      box-shadow: 0 5px 20px rgba(0,0,0,0.1);
    }

    h1 {
      font-size: 40px;
    }

    h2 {
      font-size: 28px;
    }

    p {
      font-size: 20px;
    }

    button {
      padding: 12px 25px;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      font-size: 18px;
    }

  script.js: |
    function showMessage() {
      document.getElementById("message").innerHTML =
        "Application is successfully running on Kubernetes!";
    }
```

---

# 15. IMPORTANT — WHERE DID WE CHANGE THE APPLICATION?

The application code is inside:

```text
k8s/configmap.yaml
```

We changed:

```text
index.html
style.css
script.js
```

inside the ConfigMap.

Concept:

```text
configmap.yaml
      |
      +--- index.html
      |
      +--- style.css
      |
      +--- script.js
```

---

# 16. STEP 12 — APPLY THE NEW CONFIGMAP

Run:

```bash
kubectl apply -f configmap.yaml
```

Expected:

```text
configmap/web-content configured
```

If you see:

```text
configmap/web-content created
```

that is also fine.

---

# 17. STEP 13 — CHECK THE CONFIGMAP

Run:

```bash
kubectl get configmap -n devops-demo
```

You should see:

```text
web-content
```

Now inspect it:

```bash
kubectl describe configmap web-content -n devops-demo
```

You should see:

```text
index.html
style.css
script.js
```

---

# 18. STEP 14 — UPDATE THE DEPLOYMENT

Now we must make sure Kubernetes mounts the complete application directory.

Edit:

```text
deployment.yaml
```

Use this version:

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
              mountPath: /usr/share/nginx/html

      volumes:

        - name: web-content

          configMap:
            name: web-content
```

---

# 19. IMPORTANT CHANGE IN deployment.yaml

Previously, the ConfigMap may have been mounted like this:

```yaml
mountPath: /usr/share/nginx/html/index.html
subPath: index.html
```

Now we are mounting the entire directory:

```yaml
mountPath: /usr/share/nginx/html
```

Why?

Because our application now contains:

```text
index.html
style.css
script.js
```

We want Nginx to see all three files.

The final container directory becomes:

```text
/usr/share/nginx/html/

       |
       +--- index.html
       |
       +--- style.css
       |
       +--- script.js
```

---

# 20. STEP 15 — APPLY THE DEPLOYMENT

Run:

```bash
kubectl apply -f deployment.yaml
```

Expected:

```text
deployment.apps/webapp configured
```

---

# 21. STEP 16 — CHECK ROLLOUT

Immediately run:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

Expected:

```text
deployment "webapp" successfully rolled out
```

---

# 22. STEP 17 — CHECK PODS

Run:

```bash
kubectl get pods -n devops-demo
```

You should see two Pods.

Example:

```text
webapp-7xxxxx   Running
webapp-8xxxxx   Running
```

The Pod names may be different.

---

# 23. STEP 18 — CHECK THE DEPLOYMENT

Run:

```bash
kubectl get deployment webapp -n devops-demo
```

Expected:

```text
READY
2/2
```

---

# 24. STEP 19 — OPEN THE APPLICATION

Open:

```text
http://EC2_PUBLIC_IP:30080
```

Example:

```text
http://13.233.100.10:30080
```

You should now see:

```text
DevOps Kubernetes Project

Version 3

Application code has been updated on the existing EC2 server.

[Test Application]
```

Click:

```text
Test Application
```

Expected:

```text
Application is successfully running on Kubernetes!
```

---

# 25. IMPORTANT CONCEPT

Notice what happened.

We did NOT change:

```text
EC2 IP
```

We did NOT recreate:

```text
EC2
```

We did NOT reinstall:

```text
k3s
```

We did NOT create:

```text
new Service
```

We only changed:

```text
Application Code
      |
      v
ConfigMap
      |
      v
Deployment
      |
      v
Pods
```

This is the main lesson of today's class.

---

# 26. STEP 20 — VERIFY THE APPLICATION INSIDE A POD

First get the Pod name:

```bash
kubectl get pods -n devops-demo
```

Copy one Pod name.

Example:

```text
webapp-7d8f6c9b7f-abcde
```

Run:

```bash
kubectl exec -it webapp-7d8f6c9b7f-abcde -n devops-demo -- ls -l /usr/share/nginx/html
```

You should see:

```text
index.html
script.js
style.css
```

This proves that the application files are inside the Nginx container.

---

# 27. STEP 21 — VIEW THE HTML INSIDE THE POD

Run:

```bash
kubectl exec -it POD_NAME -n devops-demo -- cat /usr/share/nginx/html/index.html
```

Replace:

```text
POD_NAME
```

with the actual Pod name.

You should see:

```html
<h1>DevOps Kubernetes Project</h1>
<h2>Version 3</h2>
```

---

# 28. STEP 22 — VIEW THE JAVASCRIPT

Run:

```bash
kubectl exec -it POD_NAME -n devops-demo -- cat /usr/share/nginx/html/script.js
```

You should see:

```javascript
function showMessage() {
  document.getElementById("message").innerHTML =
    "Application is successfully running on Kubernetes!";
}
```

---

# 29. STEP 23 — CHANGE APPLICATION TO VERSION 4

Now students will perform their own application update.

Open:

```text
configmap.yaml
```

Find:

```html
<h2>Version 3</h2>
```

Change it to:

```html
<h2>Version 4</h2>
```

Also change:

```html
Application code has been updated on the existing EC2 server.
```

to:

```html
Version 4 deployed successfully using Kubernetes.
```

---

# 30. STEP 24 — CHANGE THE BUTTON MESSAGE

Change the JavaScript message from:

```javascript
"Application is successfully running on Kubernetes!"
```

to:

```javascript
"Version 4 is running successfully!"
```

Save the file.

---

# 31. STEP 25 — APPLY THE CONFIGMAP

Run:

```bash
kubectl apply -f configmap.yaml
```

Expected:

```text
configmap/web-content configured
```

---

# 32. STEP 26 — RESTART THE DEPLOYMENT

Run:

```bash
kubectl rollout restart deployment/webapp -n devops-demo
```

Expected:

```text
deployment.apps/webapp restarted
```

Why are we doing this?

The Pods need to load the updated application configuration.

---

# 33. STEP 27 — WATCH THE ROLLOUT

Run:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

Expected:

```text
deployment "webapp" successfully rolled out
```

---

# 34. STEP 28 — WATCH POD CHANGES

Run:

```bash
kubectl get pods -n devops-demo
```

You may observe old Pods being terminated and new Pods being created.

Concept:

```text
Old Pod
   |
   | termination
   v
New Pod
   |
   v
Version 4
```

---

# 35. STEP 29 — REFRESH THE BROWSER

Open:

```text
http://EC2_PUBLIC_IP:30080
```

Refresh the page.

Expected:

```text
DevOps Kubernetes Project

Version 4

Version 4 deployed successfully using Kubernetes.

[Test Application]
```

Click the button.

Expected:

```text
Version 4 is running successfully!
```

---

# 36. STEP 30 — CHECK ROLLOUT HISTORY

Run:

```bash
kubectl rollout history deployment/webapp -n devops-demo
```

This shows Deployment revisions.

Important:

```text
Revision
   |
   v
Deployment update history
```

---

# 37. IMPORTANT THEORY — WHAT HAPPENED?

The complete flow was:

```text
Developer
   |
   | changes HTML/CSS/JS
   v
configmap.yaml
   |
   | kubectl apply
   v
ConfigMap
   |
   | rollout restart
   v
Deployment
   |
   v
New Pods
   |
   v
Nginx
   |
   v
Browser
```

---

# 38. STEP 31 — SCALE THE APPLICATION

Now increase the application to 4 Pods.

Run:

```bash
kubectl scale deployment webapp --replicas=4 -n devops-demo
```

Check:

```bash
kubectl get pods -n devops-demo
```

Expected:

```text
4 Running Pods
```

---

# 39. STEP 32 — CHECK DEPLOYMENT

Run:

```bash
kubectl get deployment webapp -n devops-demo
```

Expected:

```text
READY
4/4
```

---

# 40. STEP 33 — CHECK ALL PODS HAVE THE APPLICATION

Run:

```bash
kubectl get pods -n devops-demo
```

For each Pod, you can verify:

```bash
kubectl exec -it POD_NAME -n devops-demo -- ls -l /usr/share/nginx/html
```

Every Pod should contain:

```text
index.html
style.css
script.js
```

---

# 41. STEP 34 — SCALE BACK TO 2

Run:

```bash
kubectl scale deployment webapp --replicas=2 -n devops-demo
```

Check:

```bash
kubectl get pods -n devops-demo
```

Expected:

```text
2 Running Pods
```

---

# 42. STEP 35 — CONTROLLED TROUBLESHOOTING EXERCISE

Now we will create a simple application problem.

This is a teaching exercise.

Do NOT delete the Deployment.

Open:

```text
configmap.yaml
```

Change the HTML link:

```html
<link rel="stylesheet" href="style.css">
```

to:

```html
<link rel="stylesheet" href="wrong-style.css">
```

Save the file.

Apply:

```bash
kubectl apply -f configmap.yaml
```

Restart:

```bash
kubectl rollout restart deployment/webapp -n devops-demo
```

Check:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

The Pods may still show:

```text
Running
```

This is important.

A Pod can be Running even when the application has a problem.

---

# 43. STEP 36 — TROUBLESHOOT THE APPLICATION

Check Pods:

```bash
kubectl get pods -n devops-demo
```

Check files:

```bash
kubectl exec -it POD_NAME -n devops-demo -- ls -l /usr/share/nginx/html
```

You should see:

```text
index.html
script.js
style.css
```

But the HTML is requesting:

```text
wrong-style.css
```

while the actual file is:

```text
style.css
```

---

# 44. STEP 37 — FIND THE PROBLEM

Run:

```bash
kubectl exec -it POD_NAME -n devops-demo -- cat /usr/share/nginx/html/index.html
```

Look for:

```html
<link rel="stylesheet" href="wrong-style.css">
```

The actual file is:

```text
style.css
```

Therefore:

```text
HTML request
      |
      v
wrong-style.css
      |
      X
File does not exist
```

---

# 45. STEP 38 — FIX THE APPLICATION

Edit:

```text
configmap.yaml
```

Change:

```html
<link rel="stylesheet" href="wrong-style.css">
```

back to:

```html
<link rel="stylesheet" href="style.css">
```

Save.

Apply:

```bash
kubectl apply -f configmap.yaml
```

Restart:

```bash
kubectl rollout restart deployment/webapp -n devops-demo
```

Check:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

---

# 46. STEP 39 — VERIFY THE FIX

Open:

```text
http://EC2_PUBLIC_IP:30080
```

Refresh the browser.

The application should display correctly again.

This demonstrates:

```text
Change
  ↓
Deploy
  ↓
Test
  ↓
Find Problem
  ↓
Fix
  ↓
Deploy Again
  ↓
Verify
```

This is a basic DevOps application lifecycle.

---

# 47. STEP 40 — VIEW POD LOGS

Get Pods:

```bash
kubectl get pods -n devops-demo
```

Run:

```bash
kubectl logs POD_NAME -n devops-demo
```

Nginx request logs may appear when the browser accesses the application.

---

# 48. STEP 41 — CHECK SERVICE ENDPOINTS

Run:

```bash
kubectl get endpoints webapp-service -n devops-demo
```

This shows the backend Pod addresses associated with the Service.

Concept:

```text
Service
   |
   +--- Pod 1
   |
   +--- Pod 2
```

---

# 49. STEP 42 — CHECK LABELS

Run:

```bash
kubectl get pods -n devops-demo --show-labels
```

You should see:

```text
app=webapp
```

The Service uses:

```yaml
selector:
  app: webapp
```

The Pod uses:

```yaml
labels:
  app: webapp
```

They must match.

---

# 50. STEP 43 — CHECK THE COMPLETE APPLICATION

Run:

```bash
kubectl get all -n devops-demo
```

Students should be able to identify:

```text
Deployment
ReplicaSet
Pods
Service
```

---

# 51. APPLICATION UPDATE FLOW

Students should remember this flow:

```text
1. Change Code
       ↓
2. Update ConfigMap
       ↓
3. kubectl apply
       ↓
4. Restart / Rollout
       ↓
5. New Pods
       ↓
6. Test Application
       ↓
7. Check Logs
       ↓
8. Troubleshoot if needed
```

---

# 52. WHERE TO MAKE CHANGES

This is the most important section for students.

## Change HTML

Edit:

```text
k8s/configmap.yaml
```

Inside:

```yaml
data:
  index.html: |
```

---

## Change CSS

Edit:

```text
k8s/configmap.yaml
```

Inside:

```yaml
data:
  style.css: |
```

---

## Change JavaScript

Edit:

```text
k8s/configmap.yaml
```

Inside:

```yaml
data:
  script.js: |
```

---

## Change number of Pods

Edit:

```text
k8s/deployment.yaml
```

Change:

```yaml
replicas: 2
```

or use:

```bash
kubectl scale deployment webapp --replicas=4 -n devops-demo
```

---

## Change application port

The Service is in:

```text
k8s/service.yaml
```

Do not change ports unless you understand:

```text
containerPort
targetPort
nodePort
```

---

# 53. DO NOT CHANGE THESE TODAY

For this lab, students should NOT change:

```text
EC2 instance
k3s installation
AWS security group
Namespace name
Service name
Deployment name
```

The purpose is to learn application updates on an existing Kubernetes environment.

---

# 54. IMPORTANT DIFFERENCE — APPLICATION CODE VS INFRASTRUCTURE

## Application

```text
HTML
CSS
JavaScript
```

Managed in this exercise through:

```text
ConfigMap
```

---

## Application Runtime

```text
Nginx
Pod
Deployment
Service
```

Managed by:

```text
Kubernetes
```

---

## Infrastructure

```text
EC2
Network
Security Group
```

Managed by:

```text
AWS / Terraform
```

---

# 55. COMPLETE ARCHITECTURE

```text
                    AWS
                     |
                     v
                  EC2 SERVER
                     |
                     v
                    k3s
                     |
                     v
              Kubernetes Cluster
                     |
             +-------+-------+
             |               |
             v               v
        ConfigMap       Deployment
             |               |
             |               v
             |          ReplicaSet
             |               |
             |        +------+------+
             |        |             |
             v        v             v
        Application   Pod           Pod
        Code          |             |
                      v             v
                    Nginx         Nginx
                      |             |
                      +------+------+
                             |
                             v
                          Service
                             |
                             v
                          NodePort
                             |
                             v
                           Browser
```

---

# 56. 3-HOUR PRACTICAL CHECKPOINTS

## After Hour 1

Students should be able to:

```text
[ ] Connect to existing EC2
[ ] Check Kubernetes node
[ ] Check Pods
[ ] Check Deployment
[ ] Check Service
[ ] Open application
[ ] Explain application flow
```

---

## After Hour 2

Students should be able to:

```text
[ ] Modify HTML
[ ] Modify CSS
[ ] Modify JavaScript
[ ] Update ConfigMap
[ ] Update Deployment
[ ] Restart Pods
[ ] Check rollout
[ ] Verify application
```

---

## After Hour 3

Students should be able to:

```text
[ ] Scale application
[ ] Inspect Pod files
[ ] Check labels
[ ] Check Service endpoints
[ ] Check logs
[ ] Identify a bad application configuration
[ ] Fix the configuration
[ ] Verify the fix
```

---

# 57. MINI STUDENT CHALLENGE

Now students must create:

```text
Version 5
```

They must change all three:

### HTML

Change the heading to:

```text
DevOps Application - Version 5
```

### CSS

Change the page design.

Students can modify:

```text
font-size
padding
border-radius
background
box-shadow
```

### JavaScript

Change the button message to:

```text
Version 5 is successfully deployed!
```

---

# 58. STUDENT CHALLENGE COMMAND FLOW

Students should perform:

```bash
cd ~/kubernetes-terraform-demo/k8s
```

Edit:

```text
configmap.yaml
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

Restart:

```bash
kubectl rollout restart deployment/webapp -n devops-demo
```

Check:

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

Check Pods:

```bash
kubectl get pods -n devops-demo
```

Open:

```text
http://EC2_PUBLIC_IP:30080
```

Verify Version 5.

---

# 59. FINAL COMMAND CHEAT SHEET

## Kubernetes Status

```bash
kubectl get nodes
```

```bash
kubectl get all -n devops-demo
```

---

## Pods

```bash
kubectl get pods -n devops-demo
```

```bash
kubectl get pods -n devops-demo --show-labels
```

---

## Deployment

```bash
kubectl get deployment webapp -n devops-demo
```

```bash
kubectl describe deployment webapp -n devops-demo
```

---

## Service

```bash
kubectl get service webapp-service -n devops-demo
```

```bash
kubectl get endpoints webapp-service -n devops-demo
```

---

## ConfigMap

```bash
kubectl get configmap -n devops-demo
```

```bash
kubectl describe configmap web-content -n devops-demo
```

---

## Apply Changes

```bash
kubectl apply -f configmap.yaml
```

```bash
kubectl apply -f deployment.yaml
```

---

## Restart Application

```bash
kubectl rollout restart deployment/webapp -n devops-demo
```

---

## Rollout

```bash
kubectl rollout status deployment/webapp -n devops-demo
```

```bash
kubectl rollout history deployment/webapp -n devops-demo
```

---

## Scaling

```bash
kubectl scale deployment webapp --replicas=4 -n devops-demo
```

```bash
kubectl scale deployment webapp --replicas=2 -n devops-demo
```

---

## Logs

```bash
kubectl logs POD_NAME -n devops-demo
```

---

## Inspect Application Files

```bash
kubectl exec -it POD_NAME -n devops-demo -- ls -l /usr/share/nginx/html
```

```bash
kubectl exec -it POD_NAME -n devops-demo -- cat /usr/share/nginx/html/index.html
```

```bash
kubectl exec -it POD_NAME -n devops-demo -- cat /usr/share/nginx/html/style.css
```

```bash
kubectl exec -it POD_NAME -n devops-demo -- cat /usr/share/nginx/html/script.js
```

---

# 60. TROUBLESHOOTING QUICK FLOW

If the application is not working:

```text
1. Check Node
   |
   v
kubectl get nodes
```

Then:

```text
2. Check Pods
   |
   v
kubectl get pods -n devops-demo
```

Then:

```text
3. Check Deployment
   |
   v
kubectl get deployment -n devops-demo
```

Then:

```text
4. Check Service
   |
   v
kubectl get service -n devops-demo
```

Then:

```text
5. Check ConfigMap
   |
   v
kubectl describe configmap web-content -n devops-demo
```

Then:

```text
6. Check files inside Pod
   |
   v
kubectl exec ...
```

Then:

```text
7. Check logs
   |
   v
kubectl logs ...
```

---

# 61. WHAT STUDENTS SHOULD UNDERSTAND BY THE END

Students should be able to explain:

### Question 1

**Where did we change the application code?**

Answer:

```text
k8s/configmap.yaml
```

---

### Question 2

**Did we create a new EC2?**

Answer:

```text
No.
We reused the existing EC2 server.
```

---

### Question 3

**Why did we restart the Deployment?**

Answer:

```text
To make the running Pods load the updated application configuration.
```

---

### Question 4

**Who manages the Pods?**

Answer:

```text
Kubernetes Deployment
```

---

### Question 5

**Who provides stable access to the Pods?**

Answer:

```text
Kubernetes Service
```

---

### Question 6

**What port do we use from the browser?**

Answer:

```text
30080
```

---

### Question 7

**What is inside the Nginx web directory?**

Answer:

```text
index.html
style.css
script.js
```

---

### Question 8

**What happens when we scale from 2 to 4?**

Answer:

```text
Kubernetes creates additional Pods.
```

---

# 62. KEY DEVOPS CONCEPT

Today's most important lesson:

```text
                SAME INFRASTRUCTURE

                   AWS EC2
                      |
                      v
                     k3s
                      |
                      v
                  Kubernetes
                      |
                      v
                 Application
                      |
          +-----------+-----------+
          |           |           |
       Version 3   Version 4   Version 5
          |           |           |
          +-----------+-----------+
                      |
                      v
                 Same Service
                      |
                      v
                   Browser
```

Application code can change frequently.

The infrastructure does not need to be recreated for every application change.

---

# 63. REAL-WORLD DEVOPS CONNECTION

A real development team may work like this:

```text
Developer changes code
        |
        v
Git
        |
        v
Build / CI
        |
        v
Container Image
        |
        v
Kubernetes
        |
        v
New Version
        |
        v
Rolling Update
        |
        v
Production
```

Today's lab focuses on the Kubernetes part:

```text
Application Change
       |
       v
Kubernetes Update
       |
       v
New Pods
       |
       v
Verification
```

Later, this can be connected to:

```text
GitHub
   ↓
Jenkins
   ↓
Docker
   ↓
Kubernetes
```

---

# 64. END-OF-CLASS CHECKLIST

Students must complete:

```text
[ ] Connect to existing EC2
[ ] Verify k3s
[ ] Verify Kubernetes node
[ ] Verify existing application
[ ] Inspect Deployment
[ ] Inspect Service
[ ] Inspect ConfigMap
[ ] Create Version 3
[ ] Update HTML
[ ] Update CSS
[ ] Update JavaScript
[ ] Apply ConfigMap
[ ] Restart Deployment
[ ] Verify rollout
[ ] Open application
[ ] Scale to 4 Pods
[ ] Scale back to 2 Pods
[ ] Inspect files inside Pod
[ ] Check Service endpoints
[ ] Check logs
[ ] Perform troubleshooting exercise
[ ] Fix application
[ ] Create Version 5 challenge
```

---

# 65. FINAL FLOW TO REMEMBER

```text
                    SAME EC2
                       |
                       v
                      k3s
                       |
                       v
                 Kubernetes
                       |
                       v
                 ConfigMap
                       |
             +---------+---------+
             |         |         |
             v         v         v
          HTML       CSS       JS
             |         |         |
             +---------+---------+
                       |
                       v
                  Deployment
                       |
                 +-----+-----+
                 |           |
                 v           v
                Pod         Pod
                 |           |
                 +-----+-----+
                       |
                       v
                    Service
                       |
                       v
                     :30080
                       |
                       v
                    Browser
```

---

# END OF KUBERNETES DAY 2 LAB

**Next logical topic:**

```text
GitHub
   ↓
Docker Image Build
   ↓
Jenkins Pipeline
   ↓
Kubernetes Deployment
   ↓
Automatic Application Update
```

This will move the students from a manual Kubernetes application update to a basic CI/CD deployment workflow.
