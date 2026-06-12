# Project Overview

## What is CloudPulse AI?

CloudPulse AI is a cloud-native, microservices-based platform deployed on **Azure Kubernetes Service (AKS)**. It is designed to provide intelligent, real-time analysis of Kubernetes incidents helping teams quickly diagnose and resolve issues such as `CrashLoopBackOff`, `OOMKilled`, `ImagePullBackOff`, pod scheduling failures, and network or configuration errors.

The platform follows modern DevOps and GitOps principles:

- **Infrastructure as Code** — all Azure infrastructure is defined and managed with Terraform
- **GitOps deployments** — ArgoCD watches the GitOps repository and automatically syncs changes to AKS
- **Automated CI/CD** — GitHub Actions handles building, testing, scanning, and publishing Docker images
- **Environment parity** — identical Helm charts are used for both `dev` and `prod`, with environment-specific values files

---

## Business Value

| Problem | CloudPulse AI Solution |
|---|---|
| Kubernetes incidents are hard to diagnose manually | AI service provides root-cause analysis and remediation steps |
| Inconsistent deployments across environments | Helm + ArgoCD ensure identical, reproducible deployments |
| Lack of observability | Prometheus + Grafana provide real-time metrics and dashboards |
| Manual deployments are error-prone | Fully automated CI/CD with image tagging and GitOps updates |
| Security vulnerabilities in images | Trivy scanning in CD pipeline before deployment |

---

## Microservices Overview

### Frontend
- **Framework:** React 18 with Vite
- **Purpose:** User interface for submitting Kubernetes incidents and viewing AI-generated analysis
- **Port:** 80 (served via Nginx in container)
- **Container Image:** `cloudpulseacr12345.azurecr.io/cloudpulse-frontend`

### Backend API
- **Framework:** Java 21 + Spring Boot 3
- **Purpose:** REST API layer — receives incident reports from the frontend, proxies requests to the AI service, and exposes Prometheus metrics via `/actuator/prometheus`
- **Port:** 9090
- **Container Image:** `cloudpulseacr12345.azurecr.io/cloudpulse-backend`

### AI Service
- **Framework:** Python 3.11 + FastAPI
- **Purpose:** Rule-based intelligent engine that classifies Kubernetes incidents, determines severity, identifies root causes, and returns step-by-step remediation actions
- **Port:** 8000
- **Metrics:** Exposed at `/metrics` via `prometheus_fastapi_instrumentator`
- **Container Image:** `cloudpulseacr12345.azurecr.io/cloudpulse-ai-service`

---

## Incident Categories Supported

The AI service can analyze and provide recommendations for the following incident types:

| Incident Type | Severity | Example Symptoms |
|---|---|---|
| CrashLoopBackOff | High | Container repeatedly restarting |
| ImagePullBackOff | High | Cannot pull image from ACR |
| OOMKilled | High | Container exceeded memory limit |
| Pending / Unschedulable | High | No nodes with sufficient resources |
| Readiness Probe Failure | Medium | Pod not receiving traffic |
| ConfigMap / Secret Issues | Medium | Missing environment variables |
| High CPU Usage | Medium | Throttling under load |
| Network Policy Issues | Medium | Services cannot communicate |
| Ingress Misconfiguration | Medium | Routing not working |
| Node Not Ready | High | Cluster node is unavailable |

---

## Environments

| Environment | Git Branch | Kubernetes Namespace | Purpose |
|---|---|---|---|
| Development | `develop` | `dev` | Feature integration, testing |
| Production | `main` | `prod` | Live production workloads |

---

## Key Design Decisions

1. **Separate application and GitOps repositories** - separation of concerns between application code and deployment configuration
2. **Helm for packaging** - single chart with environment-specific values files avoids YAML duplication
3. **ArgoCD auto-sync with self-heal** - ensures the cluster always matches what is declared in Git
4. **Managed Identity authentication** - no static credentials; GitHub Actions runner and AKS use Azure Managed Identity
5. **Azure Key Vault CSI Driver** - secrets are never stored in Git; injected at runtime from Azure Key Vault
6. **Horizontal Pod Autoscaler** - backend and AI service scale automatically based on CPU utilization

---

*Next: [Solution Architecture](02-architecture.md)*
