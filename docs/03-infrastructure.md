# Infrastructure Provisioning

## Overview

All Azure infrastructure for CloudPulse AI is provisioned and managed using **Terraform**. The infrastructure is modular, version-controlled, and applied through a GitHub Actions workflow. No manual resource creation is performed in the Azure portal.

---

## Azure Resources

| Resource | Name | Purpose |
|---|---|---|
| Resource Group | `rajeevsingh` | Container for all CloudPulse resources |
| Virtual Network | `cloudpulse-vnet` | Isolated network for AKS |
| Subnet | `cloudpulse-aks-subnet` | AKS node subnet |
| Azure Kubernetes Service | `cloudpulse-aks` | Container orchestration platform |
| Azure Container Registry | `cloudpulseacr12345` | Private Docker image registry |
| Log Analytics Workspace | `cloudpulse-logs` | AKS diagnostics and logging |
| Azure Key Vault | `cloudpulse-kv` | Secrets management |
| Storage Account | `cloudpulsetfstaterajeev` | Remote Terraform state storage |

---

## Terraform Project Structure

```
terraform/
├── provider.tf               # Azure provider configuration
├── resource-group.tf         # Resource group
├── network.tf                # VNet and subnet
├── aks.tf                    # AKS cluster
├── acr.tf                    # Azure Container Registry
├── key-vault.tf              # Azure Key Vault
├── log-analytics.tf          # Log Analytics Workspace
├── terraform-backend.tf      # Remote state backend (Azure Storage)
├── variables.tf              # Input variable declarations
├── terraform.tfvars          # Variable values (non-sensitive)
├── outputs.tf                # Output values
└── modules/
    ├── networking/           # VNet + subnet module
    ├── aks/                  # AKS cluster module
    ├── acr/                  # ACR module
    ├── key-vault/            # Key Vault module
    └── log-analytics/        # Log Analytics module
```

---

## Terraform Variables

| Variable | Value | Description |
|---|---|---|
| `subscription_id` | `e25c0a93-...` | Azure Subscription ID |
| `resource_group_name` | `rajeevsingh` | Resource group name |
| `location` | `centralindia` | Azure region |
| `project_name` | `cloudpulse` | Naming prefix |
| `acr_name` | `cloudpulseacr12345` | ACR name (globally unique) |
| `aks_cluster_name` | `cloudpulse-aks` | AKS cluster name |

---

## Remote State Backend

Terraform state is stored remotely in an **Azure Storage Account** to enable team collaboration and prevent state drift.

```hcl
# terraform-backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rajeevsingh"
    storage_account_name = "cloudpulsetfstaterajeev"
    container_name       = "tfstate"
    key                  = "cloudpulse.terraform.tfstate"
  }
}
```

This ensures:
- State is not committed to Git
- Multiple CI/CD runs are safely serialized
- State can be inspected and recovered from Azure

---

## Terraform Module Details

### Networking Module
Provisions a Virtual Network and subnet for the AKS node pool. Provides network isolation and enables private cluster communication.

### AKS Module
Provisions the Azure Kubernetes Service cluster with:
- System-assigned Managed Identity for AcrPull access
- Log Analytics workspace integration for diagnostics
- Auto-scaling node pool
- Managed Identity-based authentication (no static service principal)

### ACR Module
Provisions a private Azure Container Registry. The AKS cluster's Managed Identity is granted the **AcrPull** role automatically, so no image pull secrets are needed.

### Key Vault Module
Provisions an Azure Key Vault for secret storage. The AKS workloads access secrets at runtime via the **Azure Key Vault CSI Driver** and `SecretProviderClass` secrets are never stored in Kubernetes manifests or Git.

### Log Analytics Module
Provisions a Log Analytics Workspace and connects it to AKS for container logs, node metrics, and diagnostic data.

---

## Terraform Workflow (GitHub Actions)

The Terraform workflow is manually triggered via `workflow_dispatch` with a choice of `plan` or `apply`.

```yaml
on:
  workflow_dispatch:
    inputs:
      action:
        type: choice
        options:
          - plan
          - apply
```

### Execution Steps

```
1. Developer pushes Terraform code changes
        │
        ▼
2. GitHub Actions workflow triggered manually (plan or apply)
        │
        ▼
3. az login --identity  (Managed Identity authentication)
        │
        ▼
4. terraform init       (download providers, init remote backend)
        │
        ▼
5. terraform fmt        (format check)
        │
        ▼
6. terraform validate   (validate configuration syntax)
        │
        ▼
7. terraform plan       (shows what will change)
        │
        ▼
8. terraform apply      (provisions/updates Azure resources)
        │
        ▼
9. Azure infrastructure is created or updated
        │
        ▼
10. AKS is connected to ACR (AcrPull via Managed Identity)
```

---

## AKS Cluster Configuration

| Setting | Value |
|---|---|
| Kubernetes Version | Latest stable |
| Node Pool Type | System |
| VM SKU | Standard_D2as_v5 |
| Authentication | Managed Identity |
| Network Plugin | Azure CNI |
| Log Analytics | Enabled |
| ACR Integration | Managed Identity (AcrPull) |

---

## Kubernetes Namespaces

After AKS provisioning, the following namespaces are created:

| Namespace | Purpose |
|---|---|
| `dev` | Development workloads |
| `prod` | Production workloads |
| `argocd` | ArgoCD controller |
| `monitoring` | Prometheus and Grafana |
| `ingress-nginx` | NGINX Ingress Controller |

---

## Infrastructure Screenshots

> - `terraform-workflow-success.png` - GitHub Actions Terraform workflow run succeeded
> - `azure-resource-group.png` - Azure Portal showing all resources in `rajeevsingh` resource group
> - `aks-overview.png` - AKS cluster overview from Azure Portal
> - `cloudpulse-ai-gitops\docs\images\acr-repositories.png` - ACR showing `cloudpulse-frontend`, `cloudpulse-backend`, `cloudpulse-ai-service` repositories
> - `terraform-backend-storage.png` - Azure Storage Account showing `tfstate` container

---

*Previous: [Solution Architecture](02-architecture.md) | Next: [CI/CD Workflow](04-cicd-workflow.md)*
