# Documentation Images

All screenshots referenced across the CloudPulse AI documentation.

---

## Terraform / Infrastructure

**GitHub Actions Terraform workflow successful run**
![Terraform Workflow Success](terraform-workflow-success.png)

**Azure Portal resource group `rajeevsingh` showing all resources**
![Azure Resource Group](azure-resource-group.png)

**ACR showing all three image repositories**
![ACR Repositories](acr-repositories.png)

**Azure Storage Account showing `tfstate` container**
![Terraform Backend Storage](terraform-backend-storage.png)

---

## CI/CD Workflows

**Application CI workflow run succeeded**
![CI Workflow Success](ci-workflow-success.png)

**Application CD workflow run succeeded**
![CD Workflow Success](cd-workflow-success.png)

**SonarCloud quality gate passed**
![SonarCloud Result](sonarcloud-result.png)

**ACR showing `dev-*` and `prod-*` image tags**
![ACR Image Tags](acr-image-tags.png)

**Automated commit in GitOps repo updating `values-dev.yaml`**
![GitOps Commit](gitops-commit.png)

---

## ArgoCD / GitOps

**ArgoCD `cloudpulse-dev` app Synced + Healthy**
![ArgoCD Dev Synced](argocd-dev-synced.png)

**ArgoCD `cloudpulse-prod` app Synced + Healthy**
![ArgoCD Prod Synced](argocd-prod-synced.png)

**ArgoCD monitoring app Synced**
![ArgoCD Monitoring Synced](argocd-monitoring-synced.png)

**ArgoCD application resource tree (pods, services, ingress, HPA)**
![ArgoCD App Tree](argocd-app-tree.png)

---

## Monitoring

**Prometheus Targets page showing `cloudpulse-backend` and `cloudpulse-ai-service` as UP**
![Prometheus Targets](prometheus-targets.png)

**Grafana CloudPulse AI dashboard with live data**
![Grafana Dashboard](grafana-dashboard.png)

![Grafana Dashboard](grafana-dashboard-1.png)

**`kubectl get pods -n monitoring` output**
![Monitoring Pods](monitoring-pods.png)

---

## Rollback

**Rollback workflow dispatch form**
![Rollback Workflow Dispatch](rollback-workflow-dispatch.png)

**Rollback workflow completed successfully**
![Rollback Workflow Success](rollback-workflow-success.png)

---
