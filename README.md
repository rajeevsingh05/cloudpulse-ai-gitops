# CloudPulse AI — Cloud-Native GitOps Deployment on Azure AKS

**CloudPulse AI** is a production-grade cloud-native microservices application deployed on **Azure Kubernetes Service (AKS)** using a fully automated GitOps pipeline. The platform provides real-time Kubernetes incident analysis powered by an AI service, observable through a Prometheus and Grafana monitoring stack.

---

## Repositories

| Repository | Purpose |
|---|---|
| [cloudpulse-ai](https://github.com/rajeevsingh05/cloudpulse-ai) | Application source code, Dockerfiles, CI/CD workflows, Terraform IaC |
| [cloudpulse-ai-gitops](https://github.com/rajeevsingh05/cloudpulse-ai-gitops) | Helm charts, Kubernetes manifests, ArgoCD applications, monitoring config |

---

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React (Vite) |
| Backend API | Java Spring Boot |
| AI Service | Python FastAPI |
| Container Registry | Azure Container Registry (ACR) |
| Orchestration | Azure Kubernetes Service (AKS) |
| Infrastructure as Code | Terraform |
| CI/CD | GitHub Actions |
| GitOps Controller | ArgoCD |
| Package Manager | Helm |
| Monitoring | Prometheus + Grafana (kube-prometheus-stack) |
| Code Quality | SonarCloud |
| Security Scanning | Trivy |
| Secrets Management | Azure Key Vault + CSI Driver |

---

## Documentation

| # | Document |
|---|---|
| 01 | [Project Overview](docs/01-project-overview.md) |
| 02 | [Solution Architecture](docs/02-architecture.md) |
| 03 | [Infrastructure Provisioning](docs/03-infrastructure.md) |
| 04 | [CI/CD Workflow](docs/04-cicd-workflow.md) |
| 05 | [GitOps Workflow](docs/05-gitops-workflow.md) |
| 06 | [Deployment Guide](docs/06-deployment-guide.md) |
| 07 | [Monitoring and Logging](docs/07-monitoring-logging.md) |
| 08 | [Rollback Strategy](docs/08-rollback-strategy.md) |
| 09 | [Challenges and Resolutions](docs/09-challenges-and-resolutions.md) |

---

## Repository Structure

```
cloudpulse-ai-gitops/
+-- README.md
+-- argocd/
|   +-- dev/cloudpulse-dev-app.yaml
|   +-- prod/cloudpulse-prod-app.yaml
|   +-- monitoring/kube-prometheus-stack-app.yaml
+-- helm/cloudpulse/
|   +-- Chart.yaml
|   +-- values.yaml
|   +-- values-dev.yaml
|   +-- values-prod.yaml
|   +-- templates/
+-- kubernetes/
|   +-- dev/
|   +-- prod/
+-- monitoring/
|   +-- servicemonitor-backend.yaml
|   +-- servicemonitor-ai-service.yaml
|   +-- dashboards/cloudpulse-dashboard.json
+-- docs/
```

---

## Environments

| Environment | Branch | Namespace | ArgoCD App |
|---|---|---|---|
| Development | `develop` | `dev` | `cloudpulse-dev` |
| Production | `main` | `prod` | `cloudpulse-prod` |

---

> **This repository is the single source of truth for all deployments.**
> ArgoCD continuously watches this repository and automatically syncs the desired state to AKS.
> No direct `kubectl apply` is used in production.
