# Security Framework — IAM, Service Principals, RBAC

## 1. Authentication Model

### Current (Mayank's POC)
```
Personal PAT Token → shared across all environments → NOT enterprise-grade
```

### Target (Production)
```
Service Principals (per environment) → each has its own token → isolated access
```

| Environment | Service Principal | Used By | Access |
|---|---|---|---|
| DEV | sp-dev-deployer | CI/CD pipeline, developers | Dev workspace only |
| STG | sp-stg-deployer | CI/CD pipeline | Stg workspace only |
| PROD | sp-prod-deployer | CI/CD pipeline | Prod workspace only |

---

## 2. IAM Roles

### Cross-Account Role (Databricks ↔ AWS)
```
Role: databricks-cross-account-role
Trust: Databricks control plane AWS account (414351767826 for us-east-1)
Permissions:
  - ec2:* (for cluster management)
  - s3:GetObject, s3:PutObject, s3:ListBucket (for data access)
  - iam:PassRole (to pass instance profile to clusters)
```

### Instance Profile (Cluster Nodes → S3)
```
Role: databricks-cluster-instance-profile
Permissions:
  - s3:GetObject, s3:PutObject, s3:DeleteObject on Bronze/Silver/Gold buckets
  - s3:ListBucket on Bronze/Silver/Gold buckets
  - kms:Decrypt, kms:GenerateDataKey (if KMS encryption used)
```

### CI/CD Role (GitHub Actions → AWS)
```
Role: github-actions-terraform-role
Trust: GitHub OIDC provider (recommended) OR static access keys
Permissions:
  - s3:* on terraform state bucket
  - dynamodb:* on terraform lock table
  - iam:PassRole (limited)
```

---

## 3. RBAC — Role-Based Access Control

### AD Group → Databricks Group Mapping

| AD Group | Databricks Group | Access Level |
|---|---|---|
| AD-DataEngineers | engineers | CAN_MANAGE clusters, jobs, notebooks. MANAGE secret scopes |
| AD-DataAnalysts | analysts | CAN_USE clusters. READ secret scopes. Read-only on Gold tables |
| AD-DevOps | devops-admins | Workspace admin. Full access |
| AD-DataLeads | data-leads | CAN_MANAGE jobs. Approve deployments |

### Unity Catalog Permissions

| Object | Engineers | Analysts | Service Principal |
|---|---|---|---|
| Catalog | USE CATALOG | USE CATALOG | ALL PRIVILEGES |
| Bronze Schema | ALL PRIVILEGES | — | ALL PRIVILEGES |
| Silver Schema | ALL PRIVILEGES | SELECT | ALL PRIVILEGES |
| Gold Schema | ALL PRIVILEGES | SELECT | ALL PRIVILEGES |
| Views (RLS) | SELECT | SELECT (row-filtered) | ALL PRIVILEGES |

### Row-Level Security (MVP Scope)
```sql
-- Example: Analysts can only see data for their business unit
CREATE VIEW gold_catalog.reporting.sales_filtered AS
SELECT * FROM gold_catalog.reporting.sales
WHERE business_unit = current_user_attribute('business_unit');
```

---

## 4. Secret Management

| Where | What | Purpose |
|---|---|---|
| GitHub Secrets (per env) | DATABRICKS_HOST, DATABRICKS_TOKEN | CI/CD pipeline auth |
| GitHub Secrets | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY (or OIDC) | Terraform AWS auth |
| Databricks Secret Scopes | DB connection strings, API keys | Runtime secrets for jobs |
| AWS Secrets Manager | Sensitive credentials (optional) | Centralized secret store |

### Secret Scope ACLs (from Mayank's Terraform)
```
Engineers group → MANAGE (can read/write secrets)
Analysts group  → READ (can only read secrets)
```

---

## 5. Network Security

| Control | Implementation |
|---|---|
| No public IPs on clusters | Private subnets only |
| S3 access via VPC Endpoint | No internet traversal for data |
| Security Group | Self-referencing for cluster-to-cluster, restricted outbound |
| Encryption at rest | S3 SSE-KMS, Databricks managed encryption |
| Encryption in transit | TLS enforced by Databricks |
