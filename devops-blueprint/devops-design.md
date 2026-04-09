# DevOps Design Document — DataStage to Databricks Migration

## 1. Project Context

Migrating ~500 MVP DataStage jobs (from 80,000+ total) to Databricks on AWS.
Tech stack: PySpark, SparkSQL, Python, Delta Lake, Databricks Asset Bundles, Terraform.

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPERS                                   │
│   Feature Branch → Pull Request → Code Review → Merge to main      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     GITHUB REPOSITORY                               │
│                                                                     │
│  src/              → Python/PySpark application code                │
│  resources/        → Databricks resource definitions (YAML)         │
│  databricks.yml    → Databricks Asset Bundle config                 │
│  terraform/        → Infrastructure as Code                         │
│  .github/workflows → CI/CD pipeline definitions                     │
│  tests/            → Unit & integration tests                       │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   GITHUB ACTIONS (CI/CD)                            │
│                                                                     │
│  CI Pipeline (on PR to main):                                       │
│    ✓ Lint (pylint, nbqa)                                           │
│    ✓ Unit Tests (pytest)                                           │
│    ✓ Terraform validate + plan                                     │
│    ✓ Databricks bundle validate                                    │
│                                                                     │
│  CD Pipeline (on release/tag):                                      │
│    DEV  → Terraform apply → Bundle deploy                          │
│    STG  → Terraform apply → Bundle deploy  (manual approval gate)  │
│    PROD → Terraform apply → Bundle deploy  (manual approval gate)  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS ACCOUNT                                    │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  VPC (Databricks-managed or Customer-managed)            │      │
│  │                                                          │      │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │      │
│  │  │ Databricks  │  │ Databricks  │  │ Databricks  │     │      │
│  │  │ Workspace   │  │ Workspace   │  │ Workspace   │     │      │
│  │  │   (DEV)     │  │   (STG)     │  │   (PROD)    │     │      │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │      │
│  │         │                │                │             │      │
│  │         ▼                ▼                ▼             │      │
│  │  ┌─────────────────────────────────────────────┐       │      │
│  │  │         Unity Catalog (Metastore)           │       │      │
│  │  │  Catalogs → Schemas → Tables/Views          │       │      │
│  │  └─────────────────────────────────────────────┘       │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  S3 Buckets (Data Lake)                                  │      │
│  │                                                          │      │
│  │  s3://project-bronze-<env>/  → Raw ingested data         │      │
│  │  s3://project-silver-<env>/  → Cleansed/transformed      │      │
│  │  s3://project-gold-<env>/    → Business-ready / curated  │      │
│  │  s3://terraform-state/       → Terraform remote state    │      │
│  │  s3://databricks-artifacts/  → Logs, checkpoints         │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  IAM                                                     │      │
│  │  - Service Principals (per env) for Databricks auth      │      │
│  │  - IAM Roles for S3 access, cross-account trust          │      │
│  │  - AD Group integration for RBAC                         │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  Monitoring & Logging                                    │      │
│  │  - CloudWatch dashboards + alarms                        │      │
│  │  - Databricks audit logs → S3 → CloudWatch               │      │
│  │  - SNS for alert notifications                           │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  Networking (if on-prem connectivity needed)             │      │
│  │  - AWS Direct Connect / VPN to on-prem                   │      │
│  │  - For Teradata/Oracle writeback                         │      │
│  └──────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Environment Strategy

| Environment | Purpose | Databricks Workspace | S3 Buckets | Access |
|-------------|---------|---------------------|------------|--------|
| DEV | Development & unit testing | Separate workspace | dev-bronze, dev-silver, dev-gold | Developers |
| STG (Test) | Integration testing, UAT | Separate workspace | stg-bronze, stg-silver, stg-gold | QA + Leads |
| PROD | Production workloads | Separate workspace | prod-bronze, prod-silver, prod-gold | Service Principals only |

Key principle: **Separate Databricks workspaces per environment** for blast radius isolation.

---

## 4. Repository Structure

```
project-repo/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # CI pipeline (runs on PR)
│       └── cd-release.yml            # CD pipeline (runs on release tag)
├── src/
│   ├── ingestion/                    # Ingestion framework code
│   ├── transformation/               # Transformation logic
│   ├── utils/                        # Shared utilities, logging, audit
│   └── config/                       # Metadata configs (YAML/JSON)
├── resources/
│   ├── jobs/                         # Databricks job definitions
│   ├── pipelines/                    # DLT pipeline definitions
│   └── clusters/                     # Cluster config definitions
├── terraform/
│   ├── envs/
│   │   ├── dev/                      # Dev environment root
│   │   ├── stg/                      # Stage environment root
│   │   └── prod/                     # Prod environment root
│   ├── modules/
│   │   ├── unity_catalog_baseline/   # Catalog + schema creation
│   │   ├── workspace_baseline/       # Secret scopes, cluster policies
│   │   ├── s3_buckets/               # Bronze/Silver/Gold buckets
│   │   └── iam/                      # IAM roles, service principals
│   └── backend.tf                    # Remote state config (S3 + DynamoDB)
├── tests/
│   ├── unit/                         # pytest unit tests
│   └── integration/                  # Integration tests
├── databricks.yml                    # Databricks Asset Bundle config
├── requirements.txt                  # Python dependencies
└── README.md
```

---

## 5. Branching Strategy

Using **Trunk-Based Development** (recommended for CI/CD):

```
main (protected)
  │
  ├── feature/JIRA-101-add-ingestion-template    ← developer works here
  │     └── PR → main (triggers CI)
  │
  ├── feature/JIRA-102-fix-transform-logic
  │     └── PR → main (triggers CI)
  │
  └── release/v1.0.0  ← tag from main (triggers CD)
        → deploys to DEV → STG (approval) → PROD (approval)
```

Rules:
- `main` is always deployable
- All changes go through Pull Requests
- PRs require: CI pass + at least 1 code review approval
- Releases are created via GitHub Release tags (e.g., v1.0.0)
- No direct pushes to main

---

## 6. What We Need — DevOps Tooling Summary

| Category | Tool | Purpose |
|----------|------|---------|
| Version Control | GitHub | Code repository, PR reviews |
| CI/CD | GitHub Actions | Build, test, deploy automation |
| Infrastructure as Code | Terraform | AWS + Databricks infra provisioning |
| Workload Deployment | Databricks Asset Bundles (DAB) | Deploy jobs, notebooks, pipelines |
| Secret Management | GitHub Secrets + AWS Secrets Manager | Store tokens, credentials per env |
| Monitoring | CloudWatch + Databricks UI | Dashboards, alerts, job monitoring |
| Logging | CloudWatch Logs | Centralized log aggregation |
| Artifact Storage | S3 | Terraform state, logs, checkpoints |
| Project Tracking | Jira | Sprint planning, task tracking |
| Documentation | Confluence / Markdown in repo | Architecture docs, runbooks |
| Security | IAM + Unity Catalog + AD Groups | RBAC, service principals |
