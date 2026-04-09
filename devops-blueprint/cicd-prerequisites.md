# CI/CD Prerequisites

Everything needed before the CI/CD pipeline can work.

---

## 1. GitHub Setup

| Item | Details | Status |
|------|---------|--------|
| GitHub Organization/Repo | Private repo under org account | TBD |
| Branch protection on `main` | Require PR, CI pass, 1+ reviewer | TBD |
| GitHub Environments | Create `dev`, `stg`, `prod` environments in repo settings | TBD |
| Environment protection rules | Manual approval for `stg` and `prod` | TBD |
| GitHub Actions enabled | Ensure Actions are enabled for the repo | TBD |

---

## 2. Secrets (per environment)

These must be configured in GitHub → Settings → Secrets → Environment secrets:

| Secret Name | Description | Scope |
|-------------|-------------|-------|
| DATABRICKS_HOST_DEV | Databricks workspace URL for dev | dev |
| DATABRICKS_HOST_STG | Databricks workspace URL for stg | stg |
| DATABRICKS_HOST_PROD | Databricks workspace URL for prod | prod |
| DATABRICKS_TOKEN_DEV | Service Principal token for dev | dev |
| DATABRICKS_TOKEN_STG | Service Principal token for stg | stg |
| DATABRICKS_TOKEN_PROD | Service Principal token for prod | prod |
| AWS_ACCESS_KEY_ID | AWS credentials for Terraform | all |
| AWS_SECRET_ACCESS_KEY | AWS credentials for Terraform | all |
| TF_STATE_BUCKET | S3 bucket name for Terraform state | all |
| TF_STATE_LOCK_TABLE | DynamoDB table for state locking | all |

> **Note**: For production, use OIDC-based auth (GitHub → AWS) instead of static keys.

---

## 3. AWS Prerequisites

| Item | Details |
|------|---------|
| AWS Account | Dedicated account or landing zone for this project |
| S3 Bucket for Terraform State | e.g., `s3://project-terraform-state` with versioning enabled |
| DynamoDB Table for State Lock | e.g., `terraform-lock` with `LockID` as partition key |
| S3 Buckets for Data Lake | Bronze/Silver/Gold per environment |
| IAM Roles | For Databricks cross-account access, Terraform execution |
| VPC (if customer-managed) | Subnets, NAT Gateway, Security Groups for Databricks |

---

## 4. Databricks Prerequisites

| Item | Details |
|------|---------|
| Databricks Account | Enterprise tier (for Unity Catalog, multiple workspaces) |
| Workspaces | One per environment (dev, stg, prod) |
| Unity Catalog Metastore | Shared metastore across workspaces |
| Service Principals | One per environment — used by CI/CD (no personal tokens) |
| Cluster Policies | Defined per environment (node types, autoscaling limits) |
| Secret Scopes | For storing credentials within Databricks |

---

## 5. Tooling Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Terraform | >= 1.5.x | Infrastructure provisioning |
| Databricks CLI | >= 0.200+ | Bundle validate/deploy |
| Python | 3.11+ | Application code, tests |
| pylint | latest | Code linting |
| nbqa | latest | Notebook linting |
| pytest | latest | Unit testing |
| GitHub Actions runners | GitHub-hosted (ubuntu-latest) | CI/CD execution |

---

## 6. Access Requirements

| Who | Needs Access To | Level |
|-----|----------------|-------|
| Developers | GitHub repo, DEV Databricks workspace | Read/Write |
| DevOps (you) | GitHub repo (admin), all Databricks workspaces, AWS account | Admin |
| Tech Lead | GitHub repo, all workspaces, approval gates | Admin |
| Service Principal | Databricks workspaces, S3 buckets | Programmatic |
| QA Team | STG Databricks workspace | Read + Execute |

---

## 7. Pre-flight Checklist Before First Pipeline Run

- [ ] GitHub repo created with branch protection
- [ ] GitHub environments (dev, stg, prod) configured with secrets
- [ ] AWS account accessible, IAM roles created
- [ ] S3 bucket for Terraform state created with versioning
- [ ] DynamoDB table for state locking created
- [ ] Databricks workspaces provisioned (dev, stg, prod)
- [ ] Service Principals created in Databricks, tokens generated
- [ ] Terraform modules tested locally
- [ ] databricks.yml targets configured with correct workspace hosts
- [ ] First CI run passes (lint + test + validate)
