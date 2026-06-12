# Documentation Images

Place all screenshots here as referenced in the documentation files.

## Required Screenshots

### Terraform / Infrastructure
- `terraform-workflow-success.png` — GitHub Actions Terraform workflow successful run
- `azure-resource-group.png` — Azure Portal resource group `rajeevsingh` showing all resources
- `aks-overview.png` — AKS cluster overview from Azure Portal
- `acr-repositories.png` — ACR showing all three image repositories
- `terraform-backend-storage.png` — Storage Account showing `tfstate` container

### CI/CD Workflows
- `ci-workflow-success.png` — Application CI workflow run succeeded
- `cd-workflow-success.png` — Application CD workflow run succeeded
- `sonarcloud-result.png` — SonarCloud quality gate passed
- `acr-image-tags.png` — ACR showing `dev-*` and `prod-*` image tags
- `gitops-commit.png` — Automated commit in GitOps repo (values-dev.yaml update)

### ArgoCD / GitOps
- `argocd-dev-synced.png` — ArgoCD `cloudpulse-dev` app Synced + Healthy
- `argocd-prod-synced.png` — ArgoCD `cloudpulse-prod` app Synced + Healthy
- `argocd-monitoring-synced.png` — ArgoCD monitoring app Synced
- `argocd-app-tree.png` — ArgoCD application resource tree

### Kubernetes
- `kubectl-pods-dev.png` — `kubectl get pods -n dev` output
- `kubectl-pods-prod.png` — `kubectl get pods -n prod` output
- `kubectl-svc-dev.png` — `kubectl get svc -n dev` output
- `kubectl-ingress.png` — `kubectl get ingress -n dev/prod` output
- `hpa-output.png` — `kubectl get hpa -n dev` output

### Monitoring
- `prometheus-targets.png` — Prometheus targets page showing services as UP
- `grafana-dashboard.png` — Grafana CloudPulse AI dashboard with live data
- `monitoring-pods.png` — `kubectl get pods -n monitoring` output

### Application
- `app-running.png` — CloudPulse AI frontend running in browser
- `incident-analysis.png` — AI incident analysis result in the UI

### Rollback
- `rollback-workflow-dispatch.png` — Rollback workflow dispatch form
- `rollback-workflow-success.png` — Rollback workflow completed


---

