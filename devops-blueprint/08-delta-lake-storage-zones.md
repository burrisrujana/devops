# Delta Lake Storage Zones — Design Document
### SOW ID #14 | Milestone 1 | Early Phase

---

## 1. Purpose

We are migrating data from old systems (Teradata, Oracle, DataStage) to the cloud (AWS + Databricks). This document describes how we will organize and store that data on AWS using the **Bronze → Silver → Gold** pattern.

---

## 2. What is Bronze, Silver, Gold?

Think of it like a water purification system — raw water goes through filters and comes out clean.

```
Source Systems ──► Bronze (Raw) ──► Silver (Cleaned) ──► Gold (Business-Ready)
```

| Zone | What It Stores | Example |
|------|---------------|---------|
| **Bronze** | Raw data — exact copy from the source, no changes | A CSV downloaded from a vendor, saved as-is |
| **Silver** | Cleaned data — duplicates removed, formats fixed, bad rows filtered | That CSV with duplicates removed and dates fixed |
| **Gold** | Business-ready data — aggregated, summarized, ready for reports | A summary table: "Total sales by region by month" |

---

## 3. Architecture

```
DATA SOURCES (On-Prem)
  Teradata, Oracle, SQL Server, Files, APIs
                    │
                    ▼
INGESTION LAYER
  Databricks reads data from sources using automated jobs
                    │
                    ▼
STORAGE (AWS S3)
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │   BRONZE     │────►│   SILVER     │────►│    GOLD      │
  │   Raw data   │     │   Cleaned    │     │   Reports    │
  │   As-is copy │     │   Validated  │     │   KPIs       │
  │   S3 + Delta │     │   S3 + Delta │     │   S3 + Delta │
  └──────────────┘     └──────────────┘     └──────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
  Engineers only       Engineers +            Everyone
                       Analysts              (BI, dashboards,
                                              business users)
```

---

## 4. S3 Bucket Design

Data is stored in **Amazon S3** — a cloud storage service with unlimited space. Inside S3, we create **buckets** (like folders on your computer).

### Data Buckets

For each environment (DEV, STG, PROD), we create 3 buckets — one per zone:

| Environment | Bronze Bucket | Silver Bucket | Gold Bucket |
|-------------|--------------|---------------|-------------|
| DEV (developers test here) | project-bronze-dev | project-silver-dev | project-gold-dev |
| STG (QA testing) | project-bronze-stg | project-silver-stg | project-gold-stg |
| PROD (real data, real users) | project-bronze-prod | project-silver-prod | project-gold-prod |

**Total: 9 data buckets**

Why separate per environment? So developers can't accidentally touch production data.

### Support Buckets

| Bucket | Purpose |
|--------|---------|
| project-terraform-state | Stores infrastructure configuration state |
| project-databricks-artifacts | Stores logs, temporary files, checkpoints |
| project-unity-catalog-metastore | Stores the data catalog (table of contents for all data) |

---

## 5. Folder Structure Inside Each Bucket

Data inside each bucket is organized by source system, then schema, then table name.

```
project-bronze-prod/
│
├── teradata/
│   ├── sales_schema/
│   │   ├── orders_table/
│   │   │   ├── year=2025/
│   │   │   │   ├── month=01/
│   │   │   │   │   └── data files (Parquet format)
│   │   │   │   └── month=02/
│   │   │   └── year=2024/
│   │   └── customers_table/
│   └── finance_schema/
│
├── oracle/
│   └── hr_schema/
│       └── employees_table/
│
└── files/
    └── daily_csv_feed/
```

Pattern: `s3://bucket-name/source_system/schema/table/`

This makes it easy to find any data — you always know where "orders from Teradata" lives.

---

## 6. Data Lifecycle Policies

Old data that nobody uses should move to cheaper storage to save money. S3 has different storage tiers:

| Storage Tier | Think of it as... | Cost | Access Speed |
|-------------|-------------------|------|-------------|
| S3 Standard | Files on your desk — used daily | Highest | Instant |
| S3 Intelligent-Tiering | Files in your drawer — used sometimes | Lower | Instant |
| S3 Glacier | Files in the storage room — rarely needed | Much lower | Minutes to hours |
| S3 Glacier Deep Archive | Files in offsite warehouse — almost never needed | Lowest | Hours |

### Lifecycle Rules by Zone

**Bronze (Raw Data)**

| Data Age | Action | Reason |
|----------|--------|--------|
| 0–90 days | Keep in S3 Standard | Actively used for processing |
| 90+ days | Move to Intelligent-Tiering | Used less frequently |
| 1+ year | Move to Glacier | Rarely accessed, keep for compliance |
| 7+ years | Delete or Deep Archive | Based on compliance requirements |

**Silver (Cleaned Data)**

| Data Age | Action | Reason |
|----------|--------|--------|
| 0–180 days | Keep in S3 Standard | Actively queried |
| 180+ days | Move to Intelligent-Tiering | Queried less often |
| 3+ years | Move to Glacier | Archive |

**Gold (Business-Ready Data)**

| Data Age | Action | Reason |
|----------|--------|--------|
| All data | Keep in S3 Standard | Always accessed by reports and dashboards |
| 3+ years | Move to Intelligent-Tiering | Older reports accessed less |

> These timelines should be confirmed with the client based on their compliance and data retention requirements.

---

## 7. Delta Tables

All data is stored as **Delta Tables** on S3. Delta is a special storage format that gives us:

| Feature | What It Means | Why It Matters |
|---------|--------------|---------------|
| Time Travel | See what data looked like yesterday or last month | Undo mistakes easily |
| ACID Transactions | Writes are all-or-nothing — no half-written data | Data is always consistent |
| Schema Enforcement | Rejects data that doesn't match expected format | Prevents bad data from entering |
| Versioning | Every change creates a new version | Full audit trail |
| Fast Queries | Data stored in optimized Parquet format | Reports load quickly |

### How Each Zone Uses Delta Tables

| Zone | How Data is Written | Details |
|------|-------------------|---------|
| Bronze | **Append only** — new data is added, nothing is changed or deleted | Keeps full history of everything received |
| Silver | **Upsert (Merge)** — update existing rows, insert new ones | Removes duplicates, tracks changes |
| Gold | **Overwrite or Merge** — rebuild summaries and aggregations | Optimized for fast reads and reporting |

---

## 8. Data Catalog (Unity Catalog)

Unity Catalog is like a **library catalog** — it helps people find data.

Just like a library has **Section → Shelf → Book**, we have **Catalog → Schema → Table**.

```
Unity Catalog
│
├── dev_catalog
│   ├── bronze_schema
│   │   ├── teradata_orders
│   │   ├── oracle_employees
│   │   └── csv_daily_feed
│   ├── silver_schema
│   │   ├── orders_cleaned
│   │   ├── employees_validated
│   │   └── daily_feed_deduped
│   └── gold_schema
│       ├── sales_by_region
│       ├── monthly_revenue
│       └── employee_headcount
│
├── stg_catalog
│   ├── bronze_schema
│   ├── silver_schema
│   └── gold_schema
│
└── prod_catalog
    ├── bronze_schema
    ├── silver_schema
    └── gold_schema
```

Unity Catalog gives us:
- **One place** to find all data across all environments
- **Access control** — decide who can see what
- **Audit trail** — track who accessed which data and when
- **Data lineage** — see where data came from and how it was transformed

---

## 9. Access Control

Not everyone should see all data:

| Role | Bronze (Raw) | Silver (Clean) | Gold (Reports) |
|------|-------------|----------------|----------------|
| Data Engineers | ✅ Read + Write | ✅ Read + Write | ✅ Read + Write |
| Data Analysts | ❌ No Access | ✅ Read Only | ✅ Read Only |
| BI Tools / Dashboards | ❌ No Access | ❌ No Access | ✅ Read Only |
| CI/CD (automated deployments) | ✅ Read + Write | ✅ Read + Write | ✅ Read + Write |
| Business Users / Managers | ❌ No Access | ❌ No Access | ✅ Read Only (filtered) |

Why this way?
- Bronze has raw, unvalidated data — only engineers should touch it
- Silver is cleaned but still technical — analysts can read, not modify
- Gold is the trusted source — everyone consumes from here

---

## 10. S3 Bucket Security Settings

Every bucket must have these settings enabled:

| Setting | What It Does |
|---------|-------------|
| Block Public Access | Prevents anyone on the internet from seeing the data |
| Encryption (SSE-KMS) | Encrypts data so it can't be read if stolen |
| Versioning | Keeps previous versions of files (like undo history) |
| Access Logging | Records who accessed the bucket and when |
| Bucket Policy | Restricts access to only approved IAM roles and VPC |
| Tags | Labels for tracking cost and ownership (Environment, Zone, Project) |

For PROD buckets only:

| Setting | What It Does |
|---------|-------------|
| Object Lock | Prevents data from being deleted (for compliance) |
| Cross-Region Replication | Copies data to another AWS region for disaster recovery (optional) |

---

## 11. Prerequisites

These things must be ready before we can build the storage zones:

| # | What We Need | Who Provides It | Status |
|---|---|---|---|
| 1 | AWS account access | Client / Cloud team | TBD |
| 2 | AWS region confirmed (e.g., us-east-1) | Client | TBD |
| 3 | KMS encryption key created | DevOps team | TBD |
| 4 | VPC with S3 VPC Endpoint ready | DevOps team | TBD |
| 5 | IAM roles for Databricks created | DevOps team | TBD |
| 6 | Databricks workspaces created (dev/stg/prod) | DevOps + Databricks team | TBD |
| 7 | Unity Catalog metastore created | Databricks team | TBD |
| 8 | List of source systems finalized | Data team | TBD |
| 9 | Data retention requirements confirmed | Client (compliance) | TBD |
| 10 | Naming conventions agreed by all teams | All teams | TBD |
| 11 | AD groups identified (engineers, analysts, etc.) | Client IT | TBD |
| 12 | Budget approved for S3 storage | Client / PM | TBD |

---

## 12. Implementation Steps

When we get access and approvals, here's the order we build things:

| Step | What We Do | Who | Depends On |
|------|-----------|-----|------------|
| 1 | Create KMS encryption key | DevOps | AWS account access |
| 2 | Create 9 S3 data buckets + 3 support buckets | DevOps | Step 1 |
| 3 | Apply security settings (encryption, versioning, block public access) | DevOps | Step 2 |
| 4 | Apply lifecycle rules (auto-archive old data) | DevOps | Step 2 |
| 5 | Set bucket policies (restrict to VPC + IAM roles only) | DevOps | Step 2 + IAM roles |
| 6 | Register S3 locations in Unity Catalog | Databricks team | Step 2 + Catalog ready |
| 7 | Create catalogs (dev/stg/prod) in Unity Catalog | Databricks team | Step 6 |
| 8 | Create schemas (bronze/silver/gold) in each catalog | Databricks team | Step 7 |
| 9 | Set permissions (who can access which schema) | DevOps + Databricks | Step 8 + AD groups |
| 10 | Create sample tables and test read/write | Data team | Step 9 |
| 11 | Verify lifecycle rules are working | DevOps | Step 4 |
| 12 | Document everything and hand over | DevOps | All steps done |

---

## 13. Summary

| Question | Answer |
|----------|--------|
| What are we building? | Organized cloud storage for data using Bronze → Silver → Gold pattern |
| Where does data live? | Amazon S3 buckets, stored as Delta Tables |
| How many buckets? | 9 data buckets (3 zones × 3 environments) + 3 support buckets |
| How is data organized? | By source system → schema → table, partitioned by date |
| What happens to old data? | Automatically moved to cheaper storage over time |
| Who can access what? | Engineers: everything. Analysts: Silver + Gold read-only. BI: Gold only |
| How is data secured? | Encryption, no public access, VPC-only, IAM roles, Unity Catalog permissions |
| What do we need first? | AWS account, KMS key, VPC, IAM roles, Databricks workspaces |
