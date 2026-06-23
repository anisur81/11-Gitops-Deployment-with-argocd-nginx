# GitOps Deployment with ArgoCD and NGINX

This guide provides a step-by-step walkthrough for deploying an NGINX application on Kubernetes using GitOps principles with ArgoCD.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Set Up Your GitOps Repository](#step-1-set-up-your-gitops-repository)
3. [Step 2: Create Kubernetes Manifests](#step-2-create-kubernetes-manifests)
4. [Step 3: Push to GitHub](#step-3-push-to-github)
5. [Step 4: Install and Configure ArgoCD](#step-4-install-and-configure-argocd)
6. [Step 5: Deploy NGINX via ArgoCD](#step-5-deploy-nginx-via-argocd)
7. [Step 6: Verify the Deployment](#step-6-verify-the-deployment)
8. [Step 7: Test Self-Healing](#step-7-test-self-healing)
9. [Bonus: App of Apps Pattern](#bonus-app-of-apps-pattern)
10. [Summary](#summary)

---

## Prerequisites

Before starting, ensure you have the following:

- **A running Kubernetes cluster** (EKS, GKE, OpenShift, or local like minikube) 
- **ArgoCD installed** in the cluster, with UI and CLI access 
- **kubectl configured** for your cluster 
- **A GitHub repository** to store GitOps manifests (an empty repo is sufficient) 
- **Git installed** on your local machine 

---

## Step 1: Set Up Your GitOps Repository

Create a local Git repository that will serve as the single source of truth for your application configuration .

```bash
mkdir -p argocd-gitops/apps/nginx
cd argocd-gitops
git init
```

This creates a GitOps-compliant directory structure for managing Kubernetes applications .

**Repository Structure:**
```
argocd-gitops/
└── apps/
    └── nginx/
        ├── namespace.yaml
        ├── deployment.yaml
        └── service.yaml
```

---

## Step 2: Create Kubernetes Manifests

Create declarative Kubernetes manifests that define the desired state of the NGINX application .

### 2.1 Namespace Manifest
Create `apps/nginx/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

### 2.2 Deployment Manifest
Create `apps/nginx/deployment.yaml` for an NGINX application with replicas :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argonginx-deployment
  labels:
    app: argonginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: argonginx
  template:
    metadata:
      labels:
        app: argonginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

### 2.3 Service Manifest
Create `apps/nginx/service.yaml` to expose NGINX using a LoadBalancer :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: argonginx-service
spec:
  selector:
    app: argonginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

---

## Step 3: Push to GitHub

Commit and push the manifests to GitHub so ArgoCD can continuously track and apply changes .

```bash
git add .
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
git commit -m "Initial GitOps NGINX app"
git branch -M main
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/argocd-gitops.git
git push -u origin main
```

> **⚠️ Important:** Replace `<YOUR_GITHUB_USERNAME>` with your actual GitHub username .

---

## Step 4: Install and Configure ArgoCD

### 4.1 Create ArgoCD Namespace
```bash
kubectl create namespace argocd
```

### 4.2 Apply ArgoCD Installation Manifests 
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4.3 Expose ArgoCD Server
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### 4.4 Get ArgoCD External IP 
```bash
kubectl get svc argocd-server -n argocd
```
Wait for `EXTERNAL-IP` to appear, then copy the IP address.

### 4.5 Get ArgoCD Admin Password 
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Copy this password.

### 4.6 Access ArgoCD UI
Open your browser to `http://<YOUR_ARGO_CD_EXTERNAL_IP>` and log in with:
- **Username:** `admin`
- **Password:** The password retrieved above 

---

## Step 5: Deploy NGINX via ArgoCD

### 5.1 Create the ArgoCD Application Manifest
Create `argo-app.yaml` and replace `<YOUR_GITHUB_USERNAME>` with your GitHub username :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argo-lab
spec:
  project: default
source:
  repoURL: https://github.com/anisur81/argocdnginx.git
  path: .
  targetRevision: main
destination:
  server: https://kubernetes.default.svc
  namespace: argolab
syncPolicy:
  automated:
    prune: true
    selfHeal: true

```

**Key Settings Explained:**
- **automated:** Sync keeps the cluster aligned with Git 
- **prune:** Removes resources that are deleted from Git 
- **selfHeal:** Restores the cluster if manual changes cause drift 

### 5.2 Apply the Application
```bash
kubectl apply -f argo-app.yaml
```

### 5.3 Verify in ArgoCD UI
Return to the ArgoCD UI. The `nginx-prod` application should appear and transition to **Healthy** and **Synced** .

---

## Step 6: Verify the Deployment

### 6.1 Check Pods in the Cluster 
```bash
kubectl get pods -n prod
```

**Expected Output:**
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-55d67f7b54-glhj5   1/1     Running   0          5s
nginx-55d67f7b54-sdh9h   1/1     Running   0          5s
```

### 6.2 Check Services 
```bash
kubectl get svc -n prod
```

**Expected Output:**
```
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   LoadBalancer   34.118.225.71   34.30.xx.xx   80:32019/TCP   2m11s
```

### 6.3 Access the Application
Open your browser and navigate to `http://<NGINX_EXTERNAL_IP>` to see the NGINX welcome page .

---

## Step 7: Test Self-Healing

ArgoCD's self-healing capability ensures the cluster always matches the Git-defined desired state .

### 7.1 Manually Scale the Deployment 
```bash
kubectl scale deployment nginx -n prod --replicas=1
```

### 7.2 Observe ArgoCD Auto-Correction
ArgoCD will automatically detect the drift and restore the deployment back to **2 replicas**, matching the Git-defined desired state .

---

## Bonus: App of Apps Pattern

For managing multiple applications at scale, use the **App of Apps** pattern where a parent ArgoCD Application manages multiple child applications .

### Directory Structure
```
argocd-gitops/
├── apps/
│   └── nginx/              # Child app 1 manifests
├── argocd/
│   └── apps/
│       ├── nginx-app.yaml  # Child Application definition
│       └── app-of-apps.yaml # Parent Application definition
```

### Parent Application Manifest
Create `argocd/apps/app-of-apps.yaml` :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_GITHUB_USERNAME>/argocd-gitops.git
    targetRevision: HEAD
    path: argocd/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply the parent app, and ArgoCD will automatically create and manage all child applications defined in the `argocd/apps` directory .

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| **1** | Create GitOps repository | Single source of truth established |
| **2** | Write Kubernetes manifests | Desired state defined in YAML |
| **3** | Push to GitHub | Manifests version-controlled |
| **4** | Install ArgoCD | GitOps controller in cluster |
| **5** | Deploy application via ArgoCD | NGINX deployed automatically |
| **6** | Verify deployment | Application accessible |
| **7** | Test self-healing | Cluster auto-corrects drift |

I've successfully implemented a production-grade GitOps workflow, deploying an NGINX application with automated sync, self-healing, and Git as the single source of truth .
