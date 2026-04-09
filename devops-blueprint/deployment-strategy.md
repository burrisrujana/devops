# Deployment Strategy

## 1. Deployment Model

**Hybrid Deployment**: Terraform (infra) + Databricks Asset Bundles (workloads)

```
Terraform Apply (infra first)
    → Creates catalogs, schemas, S3 buckets, IAM, cluster policies
    
Then

Databricks Asset Bundle Deploy (workloads)
    → Deploys jobs, notebooks, pipelines into the prepared environment
```

This order is critical: **infrastructure must exist before workloads are deployed**.

---

## 2. Deployment Flow

```
Developer merges PR to main
         │
         ▼
    CI Pipeline runs
    (lint, test, terraform plan, bundle validate)
         │
         ▼
    Team creates GitHub Release (e.g., v1.0.0)
         │
         ▼
    CD Pipeline triggers
         │
         ├──► DEV Environment
         │    ├── terraform init/plan/apply (dev)
         │    ├── databricks bundle validate --target dev
         │    └── databricks bundle deploy --target dev
         │
         ├──► STG Environment (manual approval gate)
         │    ├── terraform init/plan/apply (stg)
         │    ├── databricks bundle validate --target stg
         │    └── databricks bundle deploy --target stg
         │
         └──► PROD Environment (manual approval gate)
              ├── terraform init/plan/apply (prod)
              ├── databricks bundle validate --target prod
              └── databricks bundle deploy --target prod
```

---

## 3. Promotion Model

| From | To | Trigger | Gate |
|------|----|---------|------|
| PR | main | PR merge | CI pass + code review approval |
| main | DEV | Release tag | Automatic |
| DEV | STG | After DEV success | Manual approval (Tech Lead) |
| STG | PROD | After STG success + UAT pass | Manual approval (Tech Lead + Manager) |

---

## 4. Rollback Strategy

| Scenario | Rollback Method |
|----------|----------------|
| Bad code deployment | Re-deploy previous release tag (e.g., v0.9.0) |
| Terraform infra issue | `terraform plan` shows drift → apply previous state or fix forward |
| Databricks job failure | Bundle redeploy with previous tag |
| Data corruption | Restore from Delta Lake time travel (`RESTORE TABLE AS OF VERSION`) |

Steps for rollback:
1. Identify the last known good release tag
2. Create a hotfix release pointing to that tag
3. CD pipeline redeploys the previous version
4. Validate the rollback in the target environment

---

## 5. Deployment Environments Configuration

### DEV
- Auto-deploy on release
- Development mode clusters (auto-terminate after 30 min)
- Synthetic/sample data for testing
- Developers have full access

### STG (Test/UAT)
- Deploy after manual approval
- Production-like cluster configs
- Production-like data (masked if needed)
- QA team validates here

### PROD
- Deploy after manual approval + UAT sign-off
- Production clusters with autoscaling
- Real data
- Only Service Principals deploy — no human credentials
