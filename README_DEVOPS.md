# Azure AKS 3-Tier DevOps Mini Project

A simple DevOps mini project that deploys a 3-tier application to **Azure Kubernetes Service (AKS)** using **Azure DevOps Pipeline**, **Docker**, **Azure Container Registry (ACR)**, and **Bicep**.

The goal of this project is to demonstrate an end-to-end DevOps workflow:

```text
GitHub → Azure DevOps Pipeline → Docker Build → ACR → AKS
```

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Azure DevOps Pipeline Setup](#azure-devops-pipeline-setup)
- [Deployment Flow](#deployment-flow)
- [Verification](#verification)
- [CI/CD Demo](#cicd-demo)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [Future Improvements](#future-improvements)
- [Clean Up](#clean-up)

---

## Overview

This project deploys a simple 3-tier application:

| Layer    | Technology | Description                        |
| -------- | ---------- | ---------------------------------- |
| Frontend | React      | Web UI exposed to users            |
| Backend  | Express.js | API service connecting to database |
| Database | MariaDB    | Database running inside AKS        |

The application itself is intentionally simple. The main focus is the DevOps workflow: infrastructure provisioning, Docker image build, image push to registry, and Kubernetes deployment.

---

## Architecture

### Application Architecture

```text
User / Browser
      |
      v
Frontend Service - LoadBalancer
      |
      v
Backend Service - ClusterIP
      |
      v
Database Service - ClusterIP
      |
      v
MariaDB StatefulSet + PVC
```

### DevOps Architecture

```text
GitHub Repository
        |
        | Push code to main branch
        v
Azure DevOps Pipeline
        |
        | Build Docker images
        v
Azure Container Registry
        |
        | AKS pulls images
        v
Azure Kubernetes Service
```

### Key Design

- Frontend is exposed to the Internet using a Kubernetes `LoadBalancer` service.
- Backend is internal and only reachable inside the AKS cluster using `ClusterIP`.
- Database is internal and only reachable by backend using `ClusterIP`.
- MariaDB is deployed using `StatefulSet`.
- MariaDB data is stored using `PersistentVolumeClaim`.
- Database password is stored using Kubernetes `Secret`.
- Docker images are stored in Azure Container Registry.
- Infrastructure is created using Bicep.

---

## Tech Stack

| Category               | Technology               |
| ---------------------- | ------------------------ |
| Source Control         | GitHub                   |
| CI/CD                  | Azure DevOps Pipeline    |
| Cloud Platform         | Microsoft Azure          |
| Infrastructure as Code | Bicep                    |
| Containerization       | Docker                   |
| Container Registry     | Azure Container Registry |
| Kubernetes Platform    | Azure Kubernetes Service |
| Frontend               | React                    |
| Backend                | Express.js               |
| Database               | MariaDB                  |

---

## Repository Structure

```text
azure-aks-3tier-simple/
│
├── backend/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
│
├── frontend/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
│
├── db/
│
├── infra/
│   └── main.bicep
│
├── k8s/
│   └── app.yaml
│
├── azure-pipelines.yml
└── README.md
```

| Path                  | Description                         |
| --------------------- | ----------------------------------- |
| `backend/`            | Backend source code and Dockerfile  |
| `frontend/`           | Frontend source code and Dockerfile |
| `infra/main.bicep`    | Azure infrastructure definition     |
| `k8s/app.yaml`        | Kubernetes manifest                 |
| `azure-pipelines.yml` | Azure DevOps pipeline definition    |
| `README.md`           | Project documentation               |

---

## Prerequisites

Before deploying this project, make sure you have:

- Azure subscription
- Azure DevOps organization and project
- GitHub repository
- Azure CLI
- kubectl
- Docker
- Git

Login to Azure:

```powershell
az login
```

Check current subscription:

```powershell
az account show --output table
```

Set subscription if needed:

```powershell
az account set --subscription "<subscription-id>"
```

---

## Getting Started

### 1. Clone the repository

```powershell
git clone https://github.com/<your-username>/azure-aks-3tier-simple.git
cd azure-aks-3tier-simple
```

### 2. Review project variables

Open `azure-pipelines.yml` and review these variables:

```yaml
variables:
  azureServiceConnection: "sc-azure-mini3tier"

  location: "southeastasia"
  resourceGroup: "rg-mini3tier-simple"

  acrName: "acrmini3tiervan001"
  aksName: "aks-mini3tier-simple"

  backendImageName: "backend"
  frontendImageName: "frontend"

  imageTag: "$(Build.BuildId)"
  namespace: "mini3tier"
```

Important notes:

- `acrName` must be globally unique in Azure.
- ACR name must contain only lowercase letters and numbers.
- `azureServiceConnection` must match the service connection name in Azure DevOps.

Example valid ACR names:

```text
acrmini3tiervan001
acrmini3tierdemo2026
acraks3tierlab01
```

---

## Azure DevOps Pipeline Setup

### 1. Create Azure DevOps Project

Go to Azure DevOps:

```text
https://dev.azure.com
```

Create a new project, for example:

```text
mini3tier-devops
```

---

### 2. Create Azure Service Connection

In Azure DevOps:

```text
Project Settings
→ Service connections
→ New service connection
→ Azure Resource Manager
```

Recommended options for lab:

```text
Identity type: App registration (automatic)
Credential: Workload identity federation
Scope level: Subscription
```

Service connection name:

```text
sc-azure-mini3tier
```

This name must match the variable in `azure-pipelines.yml`:

```yaml
azureServiceConnection: "sc-azure-mini3tier"
```

---

### 3. Grant permissions to Service Principal

The pipeline needs permission to create Azure resources and attach ACR to AKS.

For lab/demo, assign the service principal:

```text
Contributor + User Access Administrator
```

or temporarily:

```text
Owner
```

Recommended scope:

```text
Subscription: Azure for Students
```

Why these permissions are needed:

| Permission                | Purpose                                                                            |
| ------------------------- | ---------------------------------------------------------------------------------- |
| Contributor               | Create/update Resource Group, ACR, AKS                                             |
| User Access Administrator | Create role assignment for AKS to pull images from ACR                             |
| Owner                     | Lab shortcut that includes both resource management and role assignment permission |

---

### 4. Create Pipeline from GitHub

In Azure DevOps:

```text
Pipelines
→ Create Pipeline
→ GitHub
→ Select repository
→ Existing Azure Pipelines YAML file
→ /azure-pipelines.yml
```

When creating the pipeline, Azure DevOps is authorized to access the GitHub repository. This allows Azure DevOps to:

- read source code,
- read `azure-pipelines.yml`,
- receive push events from GitHub,
- checkout the latest code during pipeline runs.

---

### 5. Add Secret Variable

In Azure DevOps Pipeline, add a secret variable:

```text
DB_PASSWORD=<your-secure-password>
```

Enable:

```text
Keep this value secret
```

This value is used to create the Kubernetes Secret for MariaDB.

---

## Deployment Flow

The pipeline is automatically triggered when code is pushed to the `main` branch.

```text
Push code to GitHub main
        |
        v
Azure DevOps receives trigger
        |
        v
Checkout latest source code
        |
        v
Deploy Infrastructure
        |
        v
Build and Push Docker Images
        |
        v
Deploy to AKS
```

### Stage 1: Deploy Infrastructure

This stage uses `AzureCLI@2` and Bicep to create or update:

- Resource Group
- Azure Container Registry
- Azure Kubernetes Service

It also attaches ACR to AKS so that AKS can pull Docker images from ACR.

Main file:

```text
infra/main.bicep
```

---

### Stage 2: Build and Push Docker Images

This stage builds Docker images for:

- `frontend`
- `backend`

Build commands use the Dockerfiles in each folder:

```bash
docker build -f backend/Dockerfile backend
docker build -f frontend/Dockerfile frontend
```

Images are tagged using the pipeline build ID:

```text
acrmini3tiervan001.azurecr.io/backend:<BuildId>
acrmini3tiervan001.azurecr.io/frontend:<BuildId>
```

Then the images are pushed to Azure Container Registry.

---

### Stage 3: Deploy to AKS

This stage:

1. Connects to AKS.
2. Creates namespace `mini3tier`.
3. Creates Kubernetes Secret for database password.
4. Renders `k8s/app.yaml`.
5. Replaces image placeholders with real image values.
6. Applies Kubernetes manifests.
7. Checks rollout status.

The Kubernetes manifest uses placeholders:

```text
__ACR_LOGIN_SERVER__
__IMAGE_TAG__
```

The pipeline replaces them with actual values before applying the manifest.

Do not apply raw `k8s/app.yaml` directly if it still contains placeholders.

---

## Manual Deployment

Normally, deployment should be done through the pipeline.

If you want to trigger deployment manually:

```powershell
git add .
git commit -m "Trigger deployment"
git push origin main
```

Then open Azure DevOps and check the latest pipeline run.

---

## Verification

### 1. Connect to AKS

```powershell
az aks get-credentials `
  --resource-group rg-mini3tier-simple `
  --name aks-mini3tier-simple `
  --admin `
  --overwrite-existing
```

---

### 2. Check AKS nodes

```powershell
kubectl get nodes
```

---

### 3. Check pods

```powershell
kubectl get pods -n mini3tier
```

Expected result:

```text
NAME                        READY   STATUS    RESTARTS   AGE
backend-xxxxx               1/1     Running   0          5m
backend-yyyyy               1/1     Running   0          5m
frontend-xxxxx              1/1     Running   0          5m
frontend-yyyyy              1/1     Running   0          5m
db-0                        1/1     Running   0          5m
```

---

### 4. Check services

```powershell
kubectl get svc -n mini3tier
```

Expected result:

```text
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
frontend   LoadBalancer   10.x.x.x        <external-ip>    80:xxxxx/TCP
backend    ClusterIP      10.x.x.x        <none>           80/TCP
db         ClusterIP      10.x.x.x        <none>           3306/TCP
```

---

### 5. Access the application

Open the frontend external IP:

```text
http://<EXTERNAL-IP>
```

If the application displays a message like:

```text
Hello from MySQL 10.6.4-MariaDB...
```

the 3-tier flow is working:

```text
Browser → Frontend → Backend → MariaDB
```

---

## Useful Commands

### Check ACR repositories

```powershell
az acr repository list `
  --name acrmini3tiervan001 `
  --output table
```

### Check image tags

```powershell
az acr repository show-tags `
  --name acrmini3tiervan001 `
  --repository frontend `
  --output table

az acr repository show-tags `
  --name acrmini3tiervan001 `
  --repository backend `
  --output table
```

### Check rollout status

```powershell
kubectl rollout status deployment/frontend -n mini3tier
kubectl rollout status deployment/backend -n mini3tier
```

### Check StatefulSet and PVC

```powershell
kubectl get statefulset -n mini3tier
kubectl get pvc -n mini3tier
```

### Check running image in AKS

```powershell
kubectl get deployment frontend -n mini3tier `
  -o jsonpath="{.spec.template.spec.containers[0].image}"

kubectl get deployment backend -n mini3tier `
  -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Expected image format:

```text
acrmini3tiervan001.azurecr.io/frontend:<BuildId>
acrmini3tiervan001.azurecr.io/backend:<BuildId>
```

### View logs

```powershell
kubectl logs deployment/frontend -n mini3tier
kubectl logs deployment/backend -n mini3tier
kubectl logs db-0 -n mini3tier
```

### View events

```powershell
kubectl get events -n mini3tier --sort-by=.lastTimestamp
```

### Describe pod

```powershell
kubectl describe pod <pod-name> -n mini3tier
```

---

## Test Backend

Backend is internal, so use port-forward.

Terminal 1:

```powershell
kubectl port-forward svc/backend 8080:80 -n mini3tier
```

Terminal 2:

```powershell
curl.exe http://127.0.0.1:8080/healthz
```

---

## Test Database

Open shell inside the MariaDB pod:

```powershell
kubectl exec -it db-0 -n mini3tier -- sh
```

Run:

```sh
mariadb -uroot -p"$(cat /etc/db-secret/db-password)" -e "SELECT VERSION(); SHOW DATABASES;"
```

Expected result:

- MariaDB version is displayed.
- Database `example` exists.

---

## CI/CD Demo

To verify CI/CD automation:

1. Edit a small text in the frontend source code.
2. Commit and push to `main`.

```powershell
git add .
git commit -m "Demo CI/CD update frontend text"
git push origin main
```

3. Open Azure DevOps Pipeline.
4. Confirm a new pipeline run starts automatically.
5. Wait until all stages succeed.
6. Refresh the application in the browser.
7. Confirm the UI has been updated.

This proves:

```text
Push code → Build image → Push to ACR → Deploy to AKS → Application updated
```

---

## Troubleshooting

### VM size is not allowed

Error:

```text
The VM size of Standard_B2s is not allowed in your subscription
```

Fix:

Use a supported VM size in `infra/main.bicep`, for example:

```text
Standard_B2s_v2
```

---

### Cannot attach ACR to AKS

Error:

```text
Could not create a role assignment for ACR. Are you an Owner on this subscription?
```

Cause:

The Azure DevOps service principal does not have permission to create role assignments.

Fix:

Grant the service principal:

```text
Contributor + User Access Administrator
```

or temporarily:

```text
Owner
```

---

### Docker build context not found

Error:

```text
path "react-express-mysql/backend" not found
```

Cause:

The pipeline build path does not match the actual repository structure.

Fix:

Use correct paths:

```bash
docker build -f backend/Dockerfile backend
docker build -f frontend/Dockerfile frontend
```

---

### InvalidImageName

Cause:

Raw `k8s/app.yaml` was applied while it still contained placeholders:

```text
__ACR_LOGIN_SERVER__
__IMAGE_TAG__
```

Fix:

Deploy through the pipeline or render the manifest before applying it manually.

---

### Secret mount error

Error:

```text
read-only file system
```

Cause:

Database secret was mounted to `/run/secrets`, which conflicts with the Kubernetes service account token path.

Fix:

Mount the secret to a separate path:

```text
/etc/db-secret
```

---

## Limitations

This project is for training and demo purposes only. It is not production-ready.

Current limitations:

- AKS is not deployed with a private network design.
- No private subnet, private endpoint, NAT Gateway, or Azure Firewall.
- Frontend is exposed directly using a public LoadBalancer.
- No HTTPS/TLS or custom domain.
- No Ingress Controller.
- No NetworkPolicy for internal traffic control.
- Secrets are stored using Kubernetes Secret instead of Azure Key Vault.
- MariaDB runs inside AKS instead of using a managed database service.
- Pipeline does not include unit tests, vulnerability scanning, or policy checks.
- Azure service connection has broad permissions for lab purposes.
- No monitoring dashboard or alerting.
- No dev/staging/production separation.

---

## Future Improvements

Possible improvements:

- Deploy AKS into a private VNet and private subnet.
- Add Ingress Controller, domain, and HTTPS/TLS.
- Use Azure Key Vault for secret management.
- Replace MariaDB StatefulSet with Azure Database for MySQL or PostgreSQL.
- Add Kubernetes NetworkPolicy.
- Add Docker image vulnerability scanning.
- Add unit and integration tests to the pipeline.
- Separate infrastructure pipeline and application deployment pipeline.
- Add Azure Monitor, Log Analytics, Prometheus, and Grafana.
- Add dev, staging, and production environments.
- Apply least privilege for Azure Service Connection and Kubernetes RBAC.

---

## Clean Up

Delete the resource group after the demo to avoid Azure cost:

```powershell
az group delete `
  --name rg-mini3tier-simple `
  --yes `
  --no-wait
```

Check resource groups:

```powershell
az group list --output table
```

---

## Summary

This project demonstrates a basic DevOps workflow for a 3-tier application on Azure.

Completed workflow:

```text
GitHub → Azure DevOps Pipeline → Docker Build → ACR → AKS
```

Main skills practiced:

- Infrastructure as Code with Bicep
- Docker image build and push
- Azure Container Registry
- Azure Kubernetes Service
- Kubernetes Deployment, Service, StatefulSet, Secret, and PVC
- CI/CD automation with Azure DevOps Pipeline
- Troubleshooting pipeline and Kubernetes deployment issues
