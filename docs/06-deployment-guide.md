# Deployment Guide

## Prerequisites

Before deploying CloudPulse AI, ensure the following are in place:

| Requirement | Details |
|---|---|
| Azure Subscription | Active subscription with Contributor access |
| Azure CLI | `az` installed and authenticated |
| Terraform | v1.5+ installed |
| kubectl | Configured for the AKS cluster |
| Helm | v3.12+ installed |
| ArgoCD CLI | `argocd` CLI installed (optional, for manual operations) |
| GitHub Actions Runner | Self-hosted runner registered to the app repository |

---

## Step 1: Clone the Repositories

```bash
# Application repository
git clone https://github.com/rajeevsingh05/cloudpulse-ai.git

# GitOps repository
git clone https://github.com/rajeevsingh05/cloudpulse-ai-gitops.git
```

---

## Step 2: Provision Azure Infrastructure

### Option A — Via GitHub Actions (Recommended)

1. Navigate to the **cloudpulse-ai** repository on GitHub
2. Go to **Actions** → **Terraform Infrastructure**
3. Click **Run workflow**
4. Select action: `plan` first to preview changes
5. Re-run with action: `apply` to provision

### Option B — Local Terraform

```bash
cd cloudpulse-ai/terraform

# Authenticate with Azure
az login

# Initialize Terraform with remote backend
terraform init

# Preview changes
terraform plan

# Apply infrastructure
terraform apply
```

After apply, resources created:

- Resource Group: `rajeevsingh`
- AKS Cluster: `cloudpulse-aks`
- ACR: `rajeevcloudpulseacr01`
- Log Analytics Workspace
- Azure Key Vault
- Virtual Network

---

## Step 3: Configure kubectl

```bash
# Get AKS credentials
az aks get-credentials \
  --resource-group rajeevsingh \
  --name cloudpulse-aks \
  --overwrite-existing

# Verify connection
kubectl get nodes
```

---

## Step 4: Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

---

## Step 5: Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## Step 6: Apply ArgoCD Applications

```bash
cd cloudpulse-ai-gitops

# Dev application
kubectl apply -f argocd/dev/cloudpulse-dev-app.yaml

# Prod application
kubectl apply -f argocd/prod/cloudpulse-prod-app.yaml

# Monitoring stack
kubectl apply -f argocd/monitoring/kube-prometheus-stack-app.yaml
```

After applying, ArgoCD will:
1. Clone the GitOps repository
2. Render the Helm chart with the appropriate values file
3. Apply all Kubernetes resources to the target namespace

---

## Step 7: Trigger the First Deployment

Push a commit to the `develop` branch in the application repository:

```bash
cd cloudpulse-ai
git checkout develop
git commit --allow-empty -m "chore: trigger initial deployment"
git push origin develop
```

This will:
1. Trigger the CI pipeline — build and test all services
2. Trigger the CD pipeline — build and push Docker images
3. Update `values-dev.yaml` with the new image tag
4. ArgoCD syncs to AKS

---

## Step 8: Apply Monitoring Resources

```bash
cd cloudpulse-ai-gitops

# ServiceMonitors for Prometheus scraping
kubectl apply -f monitoring/servicemonitor-backend.yaml
kubectl apply -f monitoring/servicemonitor-ai-service.yaml
```

---

## Verify Deployments

### Check Dev Namespace

```bash
# List all pods in dev
kubectl get pods -n dev

# Expected output:
# NAME                                        READY   STATUS    RESTARTS   AGE
# cloudpulse-frontend-xxx-xxx                 1/1     Running   0          5m
# cloudpulse-backend-xxx-xxx                  1/1     Running   0          5m
# cloudpulse-ai-service-xxx-xxx               1/1     Running   0          5m
```

### Check Prod Namespace

```bash
kubectl get pods -n prod
```

### Check Services

```bash
kubectl get svc -n dev
kubectl get svc -n prod

# Expected service types:
# NAME                      TYPE        CLUSTER-IP      PORT(S)
# cloudpulse-frontend       ClusterIP   10.0.x.x        80/TCP
# cloudpulse-backend        ClusterIP   10.0.x.x        9090/TCP
# cloudpulse-ai-service     ClusterIP   10.0.x.x        8000/TCP
```

### Check Ingress

```bash
kubectl get ingress -n dev
kubectl get ingress -n prod

# Shows the external IP / hostname for accessing the application
```

### Check HPA

```bash
kubectl get hpa -n dev
kubectl get hpa -n prod

# Expected output:
# NAME                       REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
# cloudpulse-backend-hpa     Deployment/cloudpulse-backend   10%/70%   1         3         1
# cloudpulse-ai-service-hpa  Deployment/cloudpulse-ai-service 5%/70%  1         3         1
```

### Check Monitoring Namespace

```bash
kubectl get pods -n monitoring
```

---

## Full Deployment Flow

```
Code Push to develop / main
        │
        ▼
┌─────────────────────────┐
│   CI Pipeline           │
│   ├─ Build frontend     │
│   ├─ Build backend      │
│   ├─ Build AI service   │
│   ├─ Run all tests      │
│   ├─ SonarCloud scan    │
│   └─ Docker validation  │
└────────────┬────────────┘
             │  (on success)
             ▼
┌─────────────────────────┐
│   CD Pipeline           │
│   ├─ Build Docker imgs  │
│   ├─ Push to ACR        │
│   ├─ Trivy scan         │
│   ├─ Verify in ACR      │
│   └─ Update GitOps tags │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   ArgoCD Sync           │
│   ├─ Detect git change  │
│   ├─ Render Helm chart  │
│   └─ Apply to AKS       │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   AKS Rolling Update    │
│   ├─ New pods started   │
│   ├─ Readiness checks   │
│   └─ Old pods removed   │
└────────────┬────────────┘
             │
             ▼
   Application Available via Ingress
```

---

## Accessing the Application

After deployment:

```bash
# Get Ingress external IP
kubectl get ingress -n dev -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'

# Access the application
# http://<INGRESS_IP>/          → Frontend
# http://<INGRESS_IP>/api/      → Backend API
# http://<INGRESS_IP>/ai/       → AI Service
```

---

## Useful Commands Reference

```bash
# Describe a failing pod
kubectl describe pod <pod-name> -n dev

# View pod logs
kubectl logs <pod-name> -n dev

# View previous pod logs (after crash)
kubectl logs <pod-name> -n dev --previous

# Watch pods in real-time
kubectl get pods -n dev -w

# Scale a deployment manually (emergency only)
kubectl scale deployment cloudpulse-backend -n dev --replicas=2

# Force ArgoCD sync
argocd app sync cloudpulse-dev

# Check ArgoCD app status
argocd app get cloudpulse-dev
```

---

*Previous: [GitOps Workflow](05-gitops-workflow.md) | Next: [Monitoring and Logging](07-monitoring-logging.md)*
