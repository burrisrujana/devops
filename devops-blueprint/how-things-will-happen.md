# How Things Will Happen — CI/CD Pipeline Flow

## 1. Developer Workflow (Day-to-Day)

```
1. Developer picks up a Jira ticket (e.g., JIRA-101)
2. Creates a feature branch: feature/JIRA-101-description
3. Writes code (PySpark notebooks, configs, Terraform changes)
4. Pushes branch to GitHub
5. Opens a Pull Request to main
6. CI pipeline runs automatically
7. Team reviews code + CI must pass
8. PR is merged to main
9. When ready for release → Tech Lead creates a GitHub Release tag (v1.x.x)
10. CD pipeline deploys: DEV → STG (approval) → PROD (approval)
```

---

## 2. CI Pipeline — What Happens on Every PR

Trigger: Pull Request to `main`

```yaml
Steps:
  1. Checkout code
  2. Setup Python 3.11
  3. Install dependencies (pip install -r requirements.txt)
  4. Lint
     - pylint src/          → checks Python code quality
     - nbqa pylint resources/ → checks notebook code quality
  5. Unit Tests
     - pytest tests/unit/   → runs unit tests
     - Upload test report as artifact
  6. Terraform Validation
     - terraform fmt -check -recursive  → formatting check
     - terraform init (dev)
     - terraform validate (dev)
     - terraform plan (dev)  → shows what would change (no apply!)
  7. Databricks Bundle Validation
     - databricks bundle validate --target dev
```

**CI does NOT deploy anything. It only validates.**

---

## 3. CD Pipeline — What Happens on Release

Trigger: GitHub Release published (e.g., v1.0.0)

### Job 1: Deploy to DEV (automatic)
```
1. Checkout release tag
2. Terraform init → plan → apply (dev)
3. Databricks auth check (databricks current-user me)
4. Databricks bundle validate --target dev
5. Databricks bundle deploy --target dev
```

### Job 2: Deploy to STG (requires manual approval)
```
1. Wait for DEV success
2. Manual approval from Tech Lead (GitHub Environment protection)
3. Checkout release tag
4. Terraform init → plan → apply (stg)
5. Databricks auth check
6. Databricks bundle validate --target stg
7. Databricks bundle deploy --target stg
```

### Job 3: Deploy to PROD (requires manual approval)
```
1. Wait for STG success + UAT sign-off
2. Manual approval from Tech Lead + Manager
3. Checkout release tag
4. Terraform init → plan → apply (prod)
5. Databricks auth check
6. Databricks bundle validate --target prod
7. Databricks bundle deploy --target prod
```

---

## 4. What Gets Deployed

| Component | Tool | What It Does |
|-----------|------|-------------|
| Catalogs, Schemas | Terraform | Creates/updates Unity Catalog objects |
| S3 Buckets | Terraform | Creates Bronze/Silver/Gold storage |
| IAM Roles | Terraform | Creates access roles for Databricks |
| Cluster Policies | Terraform | Defines compute governance |
| Secret Scopes | Terraform | Creates secret storage in Databricks |
| Jobs & Workflows | Databricks Asset Bundles | Deploys scheduled jobs |
| Notebooks | Databricks Asset Bundles | Deploys notebook code |
| DLT Pipelines | Databricks Asset Bundles | Deploys Delta Live Tables pipelines |

---

## 5. Approval & Gating Flow

```
PR Created
  │
  ▼
CI Passes? ──No──► PR blocked, developer fixes
  │
  Yes
  ▼
Code Review Approved? ──No──► Reviewer requests changes
  │
  Yes
  ▼
Merge to main
  │
  ▼
Release Tag Created (v1.x.x)
  │
  ▼
DEV Deploy (automatic)
  │
  ▼
DEV Validation Passes?
  │
  Yes
  ▼
Tech Lead Approves STG? ──No──► Blocked
  │
  Yes
  ▼
STG Deploy
  │
  ▼
UAT Testing Passes?
  │
  Yes
  ▼
Tech Lead + Manager Approve PROD? ──No──► Blocked
  │
  Yes
  ▼
PROD Deploy ✅
```

---

## 6. Secret Flow

```
GitHub Secrets (per environment)
  │
  ├── DATABRICKS_HOST_<ENV>  → passed as env var to workflow
  ├── DATABRICKS_TOKEN_<ENV> → passed as env var to workflow
  └── AWS credentials        → used by Terraform for S3/IAM

Terraform uses:
  TF_VAR_databricks_host  = ${{ secrets.DATABRICKS_HOST_<ENV> }}
  TF_VAR_databricks_token = ${{ secrets.DATABRICKS_TOKEN_<ENV> }}

Databricks CLI uses:
  DATABRICKS_HOST  = ${{ secrets.DATABRICKS_HOST_<ENV> }}
  DATABRICKS_TOKEN = ${{ secrets.DATABRICKS_TOKEN_<ENV> }}
```

No hardcoded credentials anywhere in code.
