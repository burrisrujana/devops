# DevOps Blueprint — Index

Complete DevOps design for DataStage to Databricks Migration on AWS.

## Documents

| # | Document | What It Covers |
|---|----------|---------------|
| 1 | [DevOps Design](01-devops-design.md) | Architecture diagram, environment strategy, repo structure, branching strategy, tooling summary |
| 2 | [Deployment Strategy](02-deployment-strategy.md) | Deployment model, promotion flow, rollback strategy, environment configs |
| 3 | [CI/CD Prerequisites](03-cicd-prerequisites.md) | GitHub setup, secrets, AWS prerequisites, Databricks prerequisites, access requirements, pre-flight checklist |
| 4 | [How Things Will Happen](04-how-things-will-happen.md) | Developer workflow, CI pipeline steps, CD pipeline steps, approval gates, secret flow |
| 5 | [Jira Board Setup](05-jira-board-setup.md) | Epics, stories, sprint plan, board columns, labels |

## Quick Reference

- **Git Platform**: GitHub
- **CI/CD Tool**: GitHub Actions
- **IaC**: Terraform
- **Workload Deployment**: Databricks Asset Bundles
- **Environments**: DEV → STG → PROD (separate workspaces)
- **Auth**: Service Principals (no personal tokens in production)
- **State Management**: S3 + DynamoDB (remote backend)
- **Monitoring**: CloudWatch + Databricks UI
