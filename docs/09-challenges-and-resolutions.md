# Challenges and Resolutions

## Overview

This document captures real technical challenges encountered during the design, implementation, and operation of CloudPulse AI. Each challenge includes the root cause, the investigation process, and the resolution applied.

---

## Challenge 1: Terraform State Corruption After Module Refactoring

### Problem

During a refactoring of Terraform code to extract inline resource definitions into reusable modules (`modules/aks`, `modules/acr`, etc.), a `terraform apply` was accidentally executed against the old (non-modular) configuration. Terraform interpreted the modular resources as new resources and the flat resources as resources to destroy.

This resulted in the **ACR and AKS integration being removed**, breaking all image pulls in the cluster (`ImagePullBackOff` on all pods).

### Root Cause

The Terraform state held references to the old flat resource addresses (e.g., `azurerm_kubernetes_cluster.aks`). After refactoring to modules, the addresses changed (e.g., `module.aks.azurerm_kubernetes_cluster.this`). Running `apply` without `terraform state mv` caused a destroy-and-recreate cycle.

### Resolution

1. Identified affected resources using `terraform plan` output and Azure Portal
2. Used `terraform state mv` to remap old resource addresses to new module paths, preserving existing Azure resources
3. Re-ran `terraform apply` — resources were updated in-place, not destroyed
4. Re-enabled AKS-to-ACR integration:
   ```bash
   az aks update \
     --name cloudpulse-aks \
     --resource-group rajeevsingh \
     --attach-acr rajeevcloudpulseacr01
   ```
5. Forced ArgoCD sync to redeploy all pods — images pulled successfully

### Prevention

- Always run `terraform plan` and review full diff before any `apply`
- When refactoring modules, use `terraform state mv` before `apply`
- Protect production infrastructure with Terraform state locking (Azure Storage Account blob lock)

---

## Challenge 2: CD Workflow Not Triggering After CI

### Problem

The CD pipeline (`application-cd.yml`) was not triggering automatically after the CI pipeline (`application-ci.yml`) completed, even when CI succeeded.

### Root Cause

The `workflow_run` trigger requires the **workflow name** in the YAML to exactly match the `name:` field of the CI workflow. A naming inconsistency — `Application CI` vs `application-ci` — caused the trigger to never fire.

Additionally, the CD workflow was initially in the same file as CI. Separating them into two files exposed a GitHub Actions limitation: `workflow_run` only triggers when the referenced workflow file is on the **default branch** (`main`). During development on `develop`, the trigger was silently skipped.

### Resolution

1. Standardized workflow names — `name: Application CI` in CI, referenced exactly in CD
2. Ensured both workflow files were committed and merged to `main` before testing the trigger chain
3. Added a `check-ci` job as the first step in CD to fail fast if `github.event.workflow_run.conclusion != "success"`:
   ```yaml
   - name: Stop if CI failed
     run: |
       if [ "${{ github.event.workflow_run.conclusion }}" != "success" ]; then
         echo "CI failed. CD will not run."
         exit 1
       fi
   ```
4. Separated CI and CD into clearly named, independently maintainable workflow files

### Prevention

- Document all workflow trigger dependencies
- Test the full CI → CD chain on a feature branch before merging to main/develop
- Use `workflow_run` only on mature, stable workflow files

---

## Challenge 3: Ingress Not Accessible Due to Company Network Restrictions

### Problem

After deploying NGINX Ingress Controller and configuring `nip.io`-based hostnames (e.g., `cloudpulse.20.1.2.3.nip.io`), the application was not accessible from the development machine.

### Root Cause

The company network blocked wildcard DNS resolution via `nip.io`. DNS queries for `*.nip.io` were intercepted and returned NXDOMAIN by the corporate DNS resolver.

### Resolution

**Dev environment:**
- Removed the hostname from the dev ingress configuration — left `ingress.host` empty in `values-dev.yaml`
- Accessed the application directly via the Ingress Controller's external IP:
  ```bash
  # Get external IP
  kubectl get svc -n ingress-nginx ingress-nginx-controller \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
  
  # Access
  http://<EXTERNAL_IP>/
  ```

**Prod environment:**
- Used a proper subdomain via Azure DNS or a custom domain
- Configured `ingress.host` in `values-prod.yaml` with the actual hostname

### Prevention

- Avoid relying on public wildcard DNS services (`nip.io`, `xip.io`) for development or CI testing
- Configure a dedicated DNS zone or use Azure DNS for proper hostname resolution
- Document network requirements for local development environments

---

## Challenge 4: Prometheus Targets Showing DOWN / No Grafana Data

### Problem

After deploying the kube-prometheus-stack, the Grafana dashboards showed "No Data" for all CloudPulse AI panels. In Prometheus under Status → Targets, the `cloudpulse-backend` and `cloudpulse-ai-service` targets were missing entirely.

### Root Cause

Two issues were found:

1. **ServiceMonitor resources were not deployed** — the `monitoring/servicemonitor-backend.yaml` and `monitoring/servicemonitor-ai-service.yaml` were not applied to the cluster
2. **Prometheus RBAC** — the kube-prometheus-stack Prometheus instance did not have permission to discover ServiceMonitors in namespaces other than `monitoring`. The `namespaceSelector` in the ServiceMonitor was correctly set to `dev` and `prod`, but the Prometheus `serviceMonitorNamespaceSelector` was not configured to allow cross-namespace discovery

### Resolution

1. Applied ServiceMonitor resources:
   ```bash
   kubectl apply -f monitoring/servicemonitor-backend.yaml
   kubectl apply -f monitoring/servicemonitor-ai-service.yaml
   ```

2. Configured the kube-prometheus-stack Helm values to allow cross-namespace ServiceMonitor discovery:
   ```yaml
   prometheus:
     prometheusSpec:
       serviceMonitorNamespaceSelector: {}    # allow all namespaces
       serviceMonitorSelector: {}             # select all ServiceMonitors
   ```

3. Verified backend exposes metrics by enabling Spring Boot Actuator:
   ```properties
   # application.properties
   management.endpoints.web.exposure.include=health,info,prometheus
   management.endpoint.prometheus.enabled=true
   ```

4. Verified AI service metrics with `curl http://localhost:8000/metrics`

5. Updated Grafana dashboard PromQL queries to use correct metric names and label selectors

### Prevention

- Include ServiceMonitor deployment as part of the initial ArgoCD application setup
- Test Prometheus scraping in the dev environment before deploying to prod
- Document required Helm values for cross-namespace ServiceMonitor discovery

---

## Challenge 5: ArgoCD Sync Failures Due to Namespace Not Existing

### Problem

When ArgoCD tried to sync `cloudpulse-dev`, it failed because the `dev` namespace did not exist yet.

### Root Cause

ArgoCD's auto-create namespace feature was not enabled. The Helm chart templates did not include a `Namespace` resource (to avoid Helm managing namespace lifecycle), and the namespace was not pre-created.

### Resolution

1. Pre-created namespaces manually before applying ArgoCD Applications:
   ```bash
   kubectl create namespace dev
   kubectl create namespace prod
   ```

2. Alternatively, enabled namespace auto-creation in ArgoCD Application:
   ```yaml
   syncPolicy:
     syncOptions:
       - CreateNamespace=true
   ```

---

## Challenge 6: Self-Hosted Runner Offline Causing Pipeline Failures

### Problem

All CI/CD pipelines failed with `Could not find any self-hosted runner matching the required labels` errors.

### Root Cause

The self-hosted GitHub Actions runner service on the VM stopped running after an OS update triggered a reboot. The runner was not configured as a system service to auto-start.

### Resolution

1. Registered the runner as a system service to ensure it auto-starts on reboot:
   ```bash
   # On the runner VM (Linux)
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

2. Added a startup verification step in critical workflows to detect offline runners early

### Prevention

- Always configure self-hosted runners as system services (`--autostart`)
- Set up monitoring/alerting on the runner VM
- Consider using Azure Container Instances or KEDA-based scale-to-zero runners for resilience

---

## Challenge 7: Image Pull Secret Not Needed but Misconfigured

### Problem

Early versions of the Helm chart included an `imagePullSecret` configuration for ACR. After migrating to Managed Identity-based AcrPull, the old secret configuration caused pods to fail with authentication errors when the Kubernetes secret did not exist.

### Root Cause

The Helm chart had `imagePullSecrets` defined in the deployment templates, but the corresponding Kubernetes secret was not created (because Managed Identity is used). Kubernetes tried to use the non-existent secret.

### Resolution

1. Removed `imagePullSecrets` from all deployment templates in the Helm chart
2. Verified AKS node pool Managed Identity had the `AcrPull` role:
   ```bash
   az aks check-acr \
     --resource-group rajeevsingh \
     --name cloudpulse-aks \
     --acr rajeevcloudpulseacr01
   ```
3. Confirmed images pull successfully without any explicit secret

---

## Summary Table

| # | Challenge | Root Cause | Resolution |
|---|---|---|---|
| 1 | Terraform state corruption | Module refactor without `state mv` | Used `state mv` + re-ran apply + fixed ACR integration |
| 2 | CD not triggering | Workflow name mismatch + branch restriction | Standardized names, merged both files to main |
| 3 | Ingress blocked by corporate network | `nip.io` DNS blocked | Used direct IP for dev, custom domain for prod |
| 4 | Prometheus targets missing | ServiceMonitors not deployed + RBAC | Applied manifests + configured cross-namespace discovery |
| 5 | ArgoCD sync namespace error | Namespace did not exist | Pre-created namespaces or used `CreateNamespace=true` |
| 6 | Self-hosted runner offline | Not configured as system service | Installed as `systemd` service with auto-start |
| 7 | imagePullSecret misconfiguration | Old config not cleaned up after Managed Identity migration | Removed `imagePullSecrets` from Helm templates |

---

*Previous: [Rollback Strategy](08-rollback-strategy.md) | Back to: [README](../README.md)*
