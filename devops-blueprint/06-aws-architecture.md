# AWS Architecture Diagram — Databricks Platform

## Architecture (Convert to Visio/PPT)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AWS ACCOUNT                                       │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        VPC (Databricks)                               │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────┐    ┌─────────────────────────┐          │  │
│  │  │   Private Subnet (AZ-a) │    │   Private Subnet (AZ-b) │          │  │
│  │  │                         │    │                         │          │  │
│  │  │  Databricks Worker      │    │  Databricks Worker      │          │  │
│  │  │  Nodes (Clusters)       │    │  Nodes (Clusters)       │          │  │
│  │  └────────────┬────────────┘    └────────────┬────────────┘          │  │
│  │               │                              │                       │  │
│  │               ▼                              ▼                       │  │
│  │  ┌──────────────────────────────────────────────────────┐           │  │
│  │  │              NAT Gateway (outbound internet)          │           │  │
│  │  └──────────────────────────┬───────────────────────────┘           │  │
│  │                             │                                       │  │
│  │  ┌──────────────────────────┴───────────────────────────┐           │  │
│  │  │           Security Groups                             │           │  │
│  │  │  - Databricks workspace SG (internal cluster comms)   │           │  │
│  │  │  - S3 VPC Endpoint SG (private S3 access)            │           │  │
│  │  └──────────────────────────────────────────────────────┘           │  │
│  │                                                                       │  │
│  │  ┌──────────────────────────────────────────────────────┐           │  │
│  │  │           VPC Endpoints                               │           │  │
│  │  │  - S3 Gateway Endpoint (private S3 access)           │           │  │
│  │  │  - STS Endpoint (for IAM auth)                       │           │  │
│  │  │  - Kinesis Endpoint (if streaming needed)            │           │  │
│  │  └──────────────────────────────────────────────────────┘           │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    DATABRICKS WORKSPACES                              │  │
│  │                                                                       │  │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐             │  │
│  │  │  DEV          │   │  STG          │   │  PROD         │             │  │
│  │  │  Workspace    │   │  Workspace    │   │  Workspace    │             │  │
│  │  │              │   │              │   │              │             │  │
│  │  │ Unity Catalog│   │ Unity Catalog│   │ Unity Catalog│             │  │
│  │  │ dev_catalog  │   │ stg_catalog  │   │ prod_catalog │             │  │
│  │  │   └─schemas  │   │   └─schemas  │   │   └─schemas  │             │  │
│  │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘             │  │
│  │         │                  │                  │                       │  │
│  │         ▼                  ▼                  ▼                       │  │
│  │  ┌──────────────────────────────────────────────────────┐           │  │
│  │  │       Unity Catalog Metastore (shared)                │           │  │
│  │  └──────────────────────────────────────────────────────┘           │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         IAM                                           │  │
│  │                                                                       │  │
│  │  Cross-Account Role ──► Databricks control plane access               │  │
│  │  Service Principal (DEV)  ──► CI/CD deploys to dev workspace          │  │
│  │  Service Principal (STG)  ──► CI/CD deploys to stg workspace          │  │
│  │  Service Principal (PROD) ──► CI/CD deploys to prod workspace         │  │
│  │  Instance Profile ──► Cluster nodes access S3                         │  │
│  │  AD Group Sync ──► Maps AD groups to Databricks groups (RBAC)         │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    S3 BUCKETS (managed by data team)                   │  │
│  │                                                                       │  │
│  │  Bronze (raw) ──► Silver (cleansed) ──► Gold (curated)                │  │
│  │  Per environment: dev / stg / prod                                    │  │
│  │  Terraform state bucket + DynamoDB lock table                         │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    MONITORING                                         │  │
│  │                                                                       │  │
│  │  CloudWatch ──► Dashboards + Alarms                                   │  │
│  │  Databricks Audit Logs ──► S3 ──► CloudWatch Logs                     │  │
│  │  SNS ──► Email/Slack alerts                                           │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    ON-PREM CONNECTIVITY (if needed)                    │  │
│  │                                                                       │  │
│  │  AWS Direct Connect / VPN ──► Teradata / Oracle on-prem               │  │
│  │  (for data writeback during migration)                                │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Checklist

| # | AWS Component | Purpose | Required? |
|---|---|---|---|
| 1 | VPC | Network isolation for Databricks clusters | Yes |
| 2 | Private Subnets (2 AZs min) | Databricks worker nodes run here | Yes |
| 3 | NAT Gateway | Outbound internet for clusters (package installs, APIs) | Yes |
| 4 | S3 Gateway VPC Endpoint | Private access to S3 (no internet needed) | Yes |
| 5 | STS VPC Endpoint | IAM authentication from within VPC | Recommended |
| 6 | Security Groups | Control traffic between clusters and to S3 | Yes |
| 7 | IAM Cross-Account Role | Databricks control plane manages resources in your AWS account | Yes |
| 8 | IAM Instance Profile | Cluster nodes use this to access S3 buckets | Yes |
| 9 | Service Principals (3) | One per env — used by CI/CD for deployments | Yes |
| 10 | S3 Buckets (Bronze/Silver/Gold) | Data lake storage (managed by data team) | Yes |
| 11 | S3 Bucket (Terraform state) | Remote backend for Terraform | Yes |
| 12 | DynamoDB Table (state lock) | Prevents concurrent Terraform runs | Yes |
| 13 | Databricks Workspaces (3) | Separate workspace per env (dev/stg/prod) | Yes |
| 14 | Unity Catalog Metastore | Shared governance layer across workspaces | Yes |
| 15 | CloudWatch | Monitoring dashboards + alarms | Yes |
| 16 | SNS | Alert notifications (email/Slack) | Yes |
| 17 | AWS Direct Connect / VPN | On-prem connectivity for Teradata/Oracle writeback | If needed |
| 18 | KMS | Encryption keys for S3 + Databricks | Recommended |

---

## Playbook — Step-by-Step Build Order

### Phase 1: AWS Foundation
```
Step 1: Create/access AWS account
Step 2: Set up VPC
         - CIDR block (e.g., 10.0.0.0/16)
         - 2 private subnets across 2 AZs
         - NAT Gateway in public subnet
         - Route tables
Step 3: Create Security Groups
         - Databricks internal SG (self-referencing for cluster comms)
         - S3 endpoint SG
Step 4: Create VPC Endpoints
         - S3 Gateway Endpoint
         - STS Interface Endpoint
Step 5: Create KMS key for encryption (optional but recommended)
```

### Phase 2: IAM Setup
```
Step 6: Create Databricks cross-account IAM role
         - Trust policy: Databricks control plane account
         - Permissions: EC2, S3, IAM PassRole
Step 7: Create IAM Instance Profile for clusters
         - S3 read/write to Bronze/Silver/Gold buckets
         - KMS decrypt/encrypt
Step 8: Create Service Principals in Databricks (one per env)
         - dev-deployer
         - stg-deployer
         - prod-deployer
Step 9: Sync AD groups to Databricks
         - Map corporate AD groups → Databricks groups
         - Engineers group, Analysts group, Admin group
```

### Phase 3: Databricks Workspaces
```
Step 10: Create Databricks workspace — DEV
          - Link to VPC + private subnets
          - Attach instance profile
Step 11: Create Databricks workspace — STG
Step 12: Create Databricks workspace — PROD
Step 13: Create Unity Catalog Metastore
          - S3 location for metastore storage
          - Assign metastore to all 3 workspaces
Step 14: Create catalogs + schemas per workspace (via Terraform)
```

### Phase 4: Terraform Backend
```
Step 15: Create S3 bucket for Terraform state
          - Enable versioning
          - Enable encryption (SSE-S3 or SSE-KMS)
          - Block public access
Step 16: Create DynamoDB table for state locking
          - Table name: terraform-lock
          - Partition key: LockID (String)
```

### Phase 5: Validation
```
Step 17: Verify Databricks workspace connectivity
Step 18: Verify cluster can launch and access S3
Step 19: Verify Service Principal can authenticate via CLI
Step 20: Verify Terraform can init/plan against each env
Step 21: Document everything — architecture diagram + this checklist
```
