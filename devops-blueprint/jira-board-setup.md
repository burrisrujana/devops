# Jira Board Setup — DevOps Track

## 1. Project Structure

Create a Jira project (Scrum board) for the DevOps track.

**Project Key suggestion**: `DEVOPS` or `DOPS`

---

## 2. Epic Breakdown

| Epic | Description | SOW IDs |
|------|-------------|---------|
| EPIC-1: AWS Infrastructure | VPC, S3, IAM, networking setup | #1, #14, #15 |
| EPIC-2: Databricks Platform | Workspace setup, Unity Catalog, cluster policies | #12, #13, #23 |
| EPIC-3: CI/CD Pipeline | GitHub Actions pipelines, approval gates, rollback | #16 |
| EPIC-4: Security & RBAC | Service Principals, AD groups, Row-Level Security | #22 |
| EPIC-5: Monitoring & Logging | CloudWatch dashboards, alerts, audit logging | #19, #20 |
| EPIC-6: Orchestration | Workflow scheduling, dependencies (Databricks/Airflow) | #18 |
| EPIC-7: Documentation & KT | Runbooks, architecture docs, self-service guide | #27, #28 |

---

## 3. Sample Stories per Epic

### EPIC-1: AWS Infrastructure
| Story | Priority | Estimate |
|-------|----------|----------|
| Design AWS architecture diagram (Visio/PPT) | High | 3d |
| Create Terraform module for S3 buckets (Bronze/Silver/Gold) | High | 2d |
| Create Terraform module for IAM roles & policies | High | 2d |
| Set up Terraform remote backend (S3 + DynamoDB) | High | 1d |
| Create VPC/networking Terraform (if customer-managed) | Medium | 3d |

### EPIC-2: Databricks Platform
| Story | Priority | Estimate |
|-------|----------|----------|
| Design Databricks workspace layout (dev/stg/prod) | High | 2d |
| Create Terraform module for Unity Catalog (catalog + schema) | High | 2d |
| Create Terraform module for cluster policies | Medium | 1d |
| Create Terraform module for secret scopes + ACLs | Medium | 1d |
| Configure databricks.yml bundle targets per env | High | 1d |

### EPIC-3: CI/CD Pipeline
| Story | Priority | Estimate |
|-------|----------|----------|
| Create CI pipeline (lint, test, terraform validate, bundle validate) | High | 3d |
| Create CD pipeline (dev → stg → prod with approval gates) | High | 3d |
| Configure GitHub environments + secrets per env | High | 1d |
| Implement rollback mechanism | Medium | 2d |
| Add Terraform state locking in pipeline | Medium | 1d |

### EPIC-4: Security & RBAC
| Story | Priority | Estimate |
|-------|----------|----------|
| Create Service Principals per environment | High | 1d |
| Design RBAC model (AD groups → Databricks groups) | High | 2d |
| Implement Row-Level Security POC | Medium | 2d |
| Document security framework | Medium | 1d |

### EPIC-5: Monitoring & Logging
| Story | Priority | Estimate |
|-------|----------|----------|
| Set up CloudWatch dashboards for pipeline health | Medium | 2d |
| Configure alerting (SNS → email/Slack) | Medium | 1d |
| Implement centralized audit logging framework | Medium | 3d |
| Databricks job monitoring dashboard | Medium | 2d |

### EPIC-6: Orchestration
| Story | Priority | Estimate |
|-------|----------|----------|
| Design orchestration framework diagram | Medium | 2d |
| Define job dependencies and scheduling strategy | Medium | 2d |
| POC: Databricks Workflows for job orchestration | Medium | 3d |

### EPIC-7: Documentation & KT
| Story | Priority | Estimate |
|-------|----------|----------|
| Write architecture runbook | Medium | 2d |
| Write CI/CD pipeline runbook | Medium | 2d |
| Write self-service DevOps guide | Medium | 3d |
| Conduct KT sessions with team | Low | 2d |

---

## 4. Sprint Plan (2-week sprints)

| Sprint | Focus | Key Deliverables |
|--------|-------|-----------------|
| Sprint 1 | Design & Planning | Architecture diagram, DevOps design doc, Jira board setup |
| Sprint 2 | AWS Infra + Terraform | S3 buckets, IAM, remote backend, Terraform modules |
| Sprint 3 | Databricks Setup | Workspaces, Unity Catalog, cluster policies, bundle config |
| Sprint 4 | CI/CD Pipeline | CI pipeline, CD pipeline, GitHub environments, secrets |
| Sprint 5 | Security + Monitoring | Service Principals, RBAC, CloudWatch, alerting |
| Sprint 6 | Orchestration + Docs | Workflow scheduling, runbooks, self-service guide, KT |

---

## 5. Board Columns

| Column | Meaning |
|--------|---------|
| Backlog | Not yet planned |
| To Do | Planned for current sprint |
| In Progress | Currently being worked on |
| In Review | PR submitted, awaiting review |
| Done | Completed and verified |

---

## 6. Labels to Use

| Label | Purpose |
|-------|---------|
| `infra` | AWS infrastructure tasks |
| `databricks` | Databricks platform tasks |
| `cicd` | CI/CD pipeline tasks |
| `security` | Security & RBAC tasks |
| `monitoring` | Monitoring & logging tasks |
| `documentation` | Docs & runbooks |
| `blocker` | Blocked — needs external input |
