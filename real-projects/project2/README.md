# Real Projects — Project 2: AWS Platform, GitHub Actions and Operations Automation

Write-ups from DevOps work on an AWS-based data and analytics platform. Infrastructure is written in OpenTofu, delivery runs on reusable GitHub Actions workflows, workloads run on ECS/Fargate and Lambda, fleets are managed with Ansible, and the data platform runs on Databricks.

> **How to talk about this work:** describe the *patterns and decisions*, for example "OIDC instead of long-lived keys" or "one directory = one state". Don't share account IDs, webhook URLs, vendor names, ticket contents or code verbatim.
>
> **Verify before claiming:** items marked 🟡 are ones where you should confirm your own role before putting them on a resume or telling them as "I built".

## Project at a glance

| Area | What was used |
|---|---|
| Cloud | AWS (12+ accounts: dev, stage, prod, QA, ingest, office…), plus Azure, GCP and Cloudflare resources |
| IaC | OpenTofu monorepo, pre-commit (`tofu fmt`, `terraform-docs`) |
| CI/CD | Reusable GitHub Actions workflows called from ~30 repositories |
| Auth to cloud | GitHub OIDC → AWS IAM roles (no static keys) |
| Compute | ECS (EC2 and Fargate), Lambda, scheduled ECS tasks |
| Containers | ECR (shared registry account), Docker, docker-compose |
| Config management | Ansible (Linux + Windows over WinRM), AWS Patch Manager |
| Security | Trivy, SAST triage, IAM / SSO audits, GitHub rulesets, AI-assisted PR review |
| Observability | Splunk alerts as code, schedule-aware SLA monitors, credential-expiry monitor, Teams notifications |
| FinOps | Daily Databricks cost report with anomaly detection; AI tool usage reporting |
| Identity | Entra ID, FusionAuth, IAM Identity Center, GitHub teams, Databricks access |
| Secrets | 1Password (SSH agent + CLI), SSM Parameter Store, GitHub secrets |
| Data platform | Databricks, Databricks Asset Bundles |

## Top 10 for interviews

| # | Topic | Interview angle | File |
|---|---|---|---|
| 1 | AI code review on every pull request | Reusable workflow, per-repo prompts, role-based security checklists | [01](01-reusable-github-actions-and-ai-pr-review.md) |
| 2 | Reusable GitHub Actions library | Platform engineering, DRY CI/CD, central governance | [01](01-reusable-github-actions-and-ai-pr-review.md) |
| 3 | GitHub OIDC to AWS | Keyless auth, trust policy scoped to repo and branch | [02](02-github-oidc-and-ecs-container-delivery.md) |
| 4 | OpenTofu plan-on-PR / apply-on-merge | GitOps IaC, one directory = one state, pre-commit | [03](03-opentofu-infrastructure-monorepo.md) |
| 5 | ECR → ECS deploy with rollout verification | Zero-downtime verification, health polling, version recorded in SSM | [02](02-github-oidc-and-ecs-container-delivery.md) |
| 6 | Credential / certificate expiry monitor | Proactive reliability, "unknown must be visible" | [04](04-observability-reliability-and-finops.md) |
| 7 | Ingest SLA monitoring | Schedule-aware alerting, dedupe and re-arm | [04](04-observability-reliability-and-finops.md) |
| 8 | Daily Databricks cost report | Cost anomaly detection, scheduled workflow, PDF email | [04](04-observability-reliability-and-finops.md) |
| 9 | Ansible fleet patching + Trivy | Check → diff → apply, secrets from 1Password | [05](05-ansible-patching-and-config-management.md) |
| 10 | Splunk alerts as code | YAML definitions, generated catalog, REST sync | [04](04-observability-reliability-and-finops.md) |

## Documents

| # | File | Topic |
|---|---|---|
| 01 | [01-reusable-github-actions-and-ai-pr-review.md](01-reusable-github-actions-and-ai-pr-review.md) | Central workflow library, AI PR review, hard-coded env gate, ticket-driven workflow |
| 02 | [02-github-oidc-and-ecs-container-delivery.md](02-github-oidc-and-ecs-container-delivery.md) | OIDC to AWS, ECR build/push, ECS deploy with rollout verification |
| 03 | [03-opentofu-infrastructure-monorepo.md](03-opentofu-infrastructure-monorepo.md) | State design, conventions, PR plan / merge apply, design docs |
| 04 | [04-observability-reliability-and-finops.md](04-observability-reliability-and-finops.md) | SLA monitor, alerts as code, credential expiry, cost reporting |
| 05 | [05-ansible-patching-and-config-management.md](05-ansible-patching-and-config-management.md) | Monthly patching, Trivy, etckeeper, Makefile UX, Patch Manager |
| 06 | [06-security-devsecops-and-compliance.md](06-security-devsecops-and-compliance.md) | CVE scanner, SAST triage story, IAM audit, rulesets |
| 07 | [07-identity-access-and-agentic-onboarding.md](07-identity-access-and-agentic-onboarding.md) | Identity scripts, multi-agent onboarding/offboarding with guardrails |
| 08 | [08-application-delivery-and-data-platform.md](08-application-delivery-and-data-platform.md) | API/frontend/agent services, Databricks Asset Bundles |
| 09 | [09-interview-quick-reference.md](09-interview-quick-reference.md) | Resume additions, gaps, 5-minute prep, quick-fire Q&A |
