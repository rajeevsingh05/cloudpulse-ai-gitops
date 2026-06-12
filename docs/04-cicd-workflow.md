# CI/CD Workflow

## Overview

CloudPulse AI uses a two-stage CI/CD pipeline implemented with **GitHub Actions**:

1. **CI (Continuous Integration)** - triggered on every push/PR; builds, tests, validates Docker images, and runs SonarCloud quality analysis
2. **CD (Continuous Delivery)** - triggered on CI success; builds and pushes Docker images to ACR, scans with Trivy, and updates image tags in the GitOps repository

The two workflows are deliberately separated to enforce the principle that **nothing is deployed unless all tests and quality gates pass**.

---

## Workflow Files

| File | Purpose |
|---|---|
| `.github/workflows/application-ci.yml` | Build, test, Docker validation, SonarCloud |
| `.github/workflows/application-cd.yml` | Image push, Trivy scan, GitOps update |
| `.github/workflows/terraform-infra.yml` | Infrastructure provisioning |
| `.github/workflows/reusable-build-test.yml` | Reusable: build all 3 services |
| `.github/workflows/reusable-docker.yml` | Reusable: Docker build (and optional push) |
| `.github/workflows/reusable-trivy.yml` | Reusable: Trivy image vulnerability scan |
| `.github/workflows/reusable-verify-acr.yml` | Reusable: Verify images exist in ACR |
| `.github/workflows/reusable-gitops-update.yml` | Reusable: Commit image tag to GitOps repo |

---

## CI Pipeline — `application-ci.yml`

### Trigger

```yaml
on:
  push:
    branches: [develop, main]
    paths:
      - "backend/**"
      - "frontend/**"
      - "ai-service/**"
  pull_request:
    branches: [develop, main]
```

CI runs only when application code or workflow files change not on GitOps or documentation changes.

### Jobs and Steps

```
┌─────────────────────────────────────┐
│  Job: prepare                       │
│  Sets env_name and image_tag        │
│  develop → dev, dev-{run_number}    │
│  main    → prod, prod-{run_number}  │
│  PR      → pr, pr-{run_number}      │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  Job: build-test                    │
│  (reusable-build-test.yml)          │
│  ├─ Build frontend (npm run build)  │
│  ├─ Build backend (mvn package)     │
│  │   └─ JaCoCo coverage            │
│  └─ Test AI service (pytest)        │
└────────────────┬────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
┌──────────────┐  ┌────────────────────────┐
│ Job: docker  │  │ Job: sonarqube          │
│ -validation  │  │ (PR + main only)        │
│              │  │                         │
│ Docker build │  │ SonarCloud scan:        │
│ (no push)    │  │ ├─ Java classes         │
│              │  │ ├─ JaCoCo coverage      │
│              │  │ ├─ AI service coverage  │
│              │  │ └─ Quality gate check   │
└──────────────┘  └─────────────────────────┘
```

### Environment Mapping

| Git Event | `env_name` | `image_tag` |
|---|---|---|
| Push to `develop` | `dev` | `dev-{run_number}` |
| Push to `main` | `prod` | `prod-{run_number}` |
| Pull Request | `pr` | `pr-{run_number}` |

---

## CD Pipeline — `application-cd.yml`

### Trigger

```yaml
on:
  workflow_run:
    workflows:
      - Application CI
    types:
      - completed
```

CD is **only** triggered when CI completes — and immediately fails if CI did not succeed.

### Jobs and Steps

```
┌──────────────────────────────────────────┐
│  Job: check-ci                           │
│  Exits with error if CI conclusion !=    │
│  "success"                               │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│  Job: prepare                            │
│  Sets env_name and image_tag from        │
│  workflow_run head_branch                │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│  Job: docker-push                        │
│  (reusable-docker.yml, push_image=true)  │
│  ├─ Build frontend Docker image          │
│  ├─ Build backend Docker image           │
│  ├─ Build AI service Docker image        │
│  └─ Push all 3 images to ACR             │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│  Job: trivy                              │
│  Scan all 3 images for vulnerabilities   │
│  Fails CD if critical CVEs found         │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│  Job: verify                             │
│  Verify all 3 images exist in ACR with   │
│  correct tag                             │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│  Job: update-gitops                      │
│  (reusable-gitops-update.yml)            │
│  ├─ Checkout cloudpulse-ai-gitops repo   │
│  ├─ Update values-{env}.yaml image tag   │
│  └─ Commit and push to GitOps repo       │
└──────────────────────────────────────────┘
         │
         ▼
   ArgoCD detects change → syncs to AKS
```

### Image Tag Format

| Environment | Image Tag Example |
|---|---|
| Development | `dev-42` |
| Production | `prod-17` |

Each image tag is unique per run, ensuring rollback to any previous version is trivial.

---

## Docker Images

All three services are built and pushed to ACR with the same tag per run:

| Service | ACR Repository |
|---|---|
| Frontend | `rajeevcloudpulseacr01.azurecr.io/cloudpulse-frontend` |
| Backend | `rajeevcloudpulseacr01.azurecr.io/cloudpulse-backend` |
| AI Service | `rajeevcloudpulseacr01.azurecr.io/cloudpulse-ai-service` |

---

## SonarCloud Code Quality

SonarCloud runs on:
- Every pull request targeting `develop` or `main`
- Every push to `main`

**Inputs:**
- Backend compiled classes (`backend/target/classes/`)
- JaCoCo XML coverage report (`backend/target/site/jacoco/jacoco.xml`)
- AI service coverage (pytest-cov)
- Frontend ESLint results

**Quality Gate:** Build fails if the SonarCloud quality gate does not pass.

---

## Terraform Workflow — `terraform-infra.yml`

Triggered manually via `workflow_dispatch`:

```
1. Developer selects "plan" or "apply"
2. az login --identity  (Managed Identity — no static credentials)
3. terraform init
4. terraform fmt
5. terraform validate
6. terraform plan   (if action = plan)
   terraform apply  (if action = apply)
```

---

## Self-Hosted Runner

All application CI/CD and Terraform jobs run on a **self-hosted runner** registered to the GitHub organization. This enables:
- Access to Azure resources via Managed Identity without credentials
- Faster builds (no cold start)
- Ability to run `kubectl`, `az`, `terraform`, `mvn`, and other tools without setup steps

---

## Branch Strategy

| Branch | Environment | Auto-deploy |
|---|---|---|
| `develop` | dev | Yes (on CI success) |
| `main` | prod | Yes (on CI success) |
| feature/* | — | CI only (no image push) |
| pull_request | — | CI + SonarCloud (no push) |

---

## Screenshots

> - `ci-workflow-success.png` — GitHub Actions CI workflow successful run
> - `cd-workflow-success.png` — GitHub Actions CD workflow successful run
> - `sonarcloud-result.png` — SonarCloud quality gate passed
> - `acr-image-tags.png` — ACR showing `dev-*` and `prod-*` image tags
> - `gitops-commit.png` — Automated commit in GitOps repo updating `values-dev.yaml`

---

*Previous: [Infrastructure Provisioning](03-infrastructure.md) | Next: [GitOps Workflow](05-gitops-workflow.md)*
