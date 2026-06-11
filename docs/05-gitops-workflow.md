# GitOps Workflow

## What is GitOps?

GitOps is the practice of using **Git as the single source of truth** for the desired state of your infrastructure and applications. Instead of running `kubectl apply` or `helm upgrade` manually, you commit changes to a Git repository and a controller (ArgoCD) ensures the cluster always matches what is declared in Git.

**Core principle:** If it's not in Git, it doesn't exist in the cluster.

---

## GitOps Architecture

```
Developer / CI Pipeline
        │
        │  git commit (image tag update)
        ▼
cloudpulse-ai-gitops (GitHub)
        │
        │  watches (polling / webhook)
        ▼
     ArgoCD
        │
        │  helm template | kubectl apply
        ▼
Azure Kubernetes Service
  ├── dev namespace
  └── prod namespace
```

---

## ArgoCD Applications

Three ArgoCD `Application` resources manage the full lifecycle of CloudPulse AI:

### 1. cloudpulse-dev

```yaml
# argocd/dev/cloudpulse-dev-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudpulse-dev
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/rajeevsingh05/cloudpulse-ai-gitops.git
    targetRevision: develop
    path: helm/cloudpulse
    helm:
      valueFiles:
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

| Property | Value |
|---|---|
| Git Branch | `develop` |
| Helm Values | `values-dev.yaml` |
| Target Namespace | `dev` |
| Auto Sync | Enabled |
| Self Heal | Enabled |
| Prune | Enabled |

### 2. cloudpulse-prod

```yaml
# argocd/prod/cloudpulse-prod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudpulse-prod
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/rajeevsingh05/cloudpulse-ai-gitops.git
    targetRevision: main
    path: helm/cloudpulse
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

| Property | Value |
|---|---|
| Git Branch | `main` |
| Helm Values | `values-prod.yaml` |
| Target Namespace | `prod` |
| Auto Sync | Enabled |
| Self Heal | Enabled |

### 3. kube-prometheus-stack

```yaml
# argocd/monitoring/kube-prometheus-stack-app.yaml
```

Deploys the full Prometheus + Grafana monitoring stack into the `monitoring` namespace via the official Helm chart.

---

## Helm Chart Structure

The single `helm/cloudpulse` chart is used for both environments. Environment differences are expressed through values files.

```
helm/cloudpulse/
├── Chart.yaml            # Chart metadata
├── values.yaml           # Base defaults (used as fallback)
├── values-dev.yaml       # Dev overrides (image tags, replicas, etc.)
├── values-prod.yaml      # Prod overrides
└── templates/
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── ai-service-deployment.yaml
    ├── ai-service-service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── secretproviderclass.yaml
    ├── hpa.yaml
    └── ingress.yaml
```

---

## Values File Hierarchy

ArgoCD renders the chart with `values.yaml` as the base, merged with the environment-specific override:

```
values.yaml  (base)
    +
values-dev.yaml  OR  values-prod.yaml  (environment override)
    =
Final rendered Kubernetes manifests
```

### Example: Dev Values (`values-dev.yaml`)

```yaml
namespace: dev

frontend:
  replicaCount: 1
  image:
    tag: dev-29          # ← Updated by CD pipeline on each deployment

backend:
  replicaCount: 1
  image:
    tag: dev-29

aiService:
  replicaCount: 1
  image:
    tag: dev-29

config:
  environment: dev

keyVault:
  secretName: cloudpulse-dev-app-secret
```

The CD pipeline commits a new `image.tag` value here. ArgoCD detects the commit and triggers a rolling deployment automatically.

---

## Sync Behavior

| Behavior | Description |
|---|---|
| **Auto Sync** | ArgoCD polls the GitOps repo every 3 minutes (default). Any new commit triggers a sync. |
| **Self Heal** | If someone manually changes a resource in AKS (e.g., `kubectl edit`), ArgoCD reverts it to match Git within the next sync cycle. |
| **Prune** | Resources that exist in the cluster but are removed from Helm templates are automatically deleted. |

---

## Deployment Flow (GitOps)

```
1. Developer merges PR to `develop` or pushes to `main`
        │
2. CI pipeline runs — builds and tests all services
        │
3. CD pipeline runs — builds Docker images and pushes to ACR
        │
4. CD pipeline updates image tag in values-{env}.yaml
   e.g.,  aiService.image.tag: dev-28  →  dev-29
        │
5. Commit pushed to cloudpulse-ai-gitops repository
        │
6. ArgoCD detects new commit (within ~3 minutes)
        │
7. ArgoCD runs: helm template | diff | apply
        │
8. Kubernetes performs rolling deployment
   - New pods are started
   - Old pods terminated after readiness checks pass
        │
9. Application is live with new image version
```

---

## Drift Detection

If a resource in AKS is **manually modified** (direct `kubectl` change), ArgoCD marks the application as **OutOfSync** and reverts the drift on the next sync cycle.

This is the **self-heal** behavior — the cluster is always forced back to the state declared in Git.

> ArgoCD self-heal is not a rollback. Self-heal restores the current Git state. Rollback is performed by reverting the image tag in Git. See [Rollback Strategy](08-rollback-strategy.md).

---

## Multi-Environment Isolation

| Aspect | Dev | Prod |
|---|---|---|
| Git Branch | `develop` | `main` |
| Namespace | `dev` | `prod` |
| ArgoCD App | `cloudpulse-dev` | `cloudpulse-prod` |
| Image Tags | `dev-*` | `prod-*` |
| Key Vault Secret | `cloudpulse-dev-app-secret` | `cloudpulse-prod-app-secret` |
| HPA Max Replicas | 3 | 5 (configurable) |

Workloads in `dev` and `prod` are completely isolated at the namespace level. A broken deployment in `dev` cannot affect `prod`.

---

## Screenshots to Add

> Add the following screenshots to `docs/images/`:
>
> - `argocd-dev-synced.png` — ArgoCD showing `cloudpulse-dev` app in Synced state
> - `argocd-prod-synced.png` — ArgoCD showing `cloudpulse-prod` app in Synced state
> - `argocd-monitoring-synced.png` — ArgoCD showing monitoring app synced
> - `argocd-app-tree.png` — ArgoCD application resource tree (pods, services, ingress, HPA)
> - `helm-values-dev.png` — `values-dev.yaml` showing updated image tag after CD run

---

*Previous: [CI/CD Workflow](04-cicd-workflow.md) | Next: [Deployment Guide](06-deployment-guide.md)*
