# 09 — Interview Quick Reference

## Suggested resume additions (only where true for you)

1. Built a reusable GitHub Actions platform and AI-assisted PR security review across ~30 repositories, with repo-specific, role-based prompts.
2. OpenTofu monorepo with plan-on-PR and apply-on-merge, one-state-per-directory conventions, and pre-commit `tofu fmt` / `terraform-docs`.
3. GitHub OIDC to AWS replacing static access keys (parallels Azure managed identity / workload identity federation).
4. ECS deploy verification: rollout status on the new task definition, health-endpoint version polling, and version tracking in SSM.
5. Credential and certificate expiry monitoring with heartbeat and `UNKNOWN` semantics.
6. FinOps: daily Databricks cost report and idle-compute root-cause analysis. Add a number only if you can defend it.
7. Ansible monthly patching with Trivy scanning and 1Password-backed secrets.
8. Agentic onboarding/offboarding with human-approval guardrails (only if you contributed).

## Gaps to be aware of

- The resume headlines **Azure, AKS, Bicep and Azure DevOps** (project1). This project is **AWS, OpenTofu, GitHub Actions, Ansible and Databricks**. Be ready to explain where each stack fits so the two stories line up.
- There is **no Kubernetes or Helm** in this project. For AKS questions, use project1.
- Some components are documented as **pending or not yet deployed** (parts of cost observability and the email intake service). Don't present them as production unless they have shipped.

## Azure ↔ AWS translation table

| Concept | Project1 (Azure) | Project2 (AWS) |
|---|---|---|
| Keyless pipeline auth | Workload identity federation + managed identity | GitHub OIDC + IAM role trust policy |
| IaC | Bicep + Deployment Stacks | OpenTofu, one state per directory |
| Plan / preview | `what-if` | `tofu plan` posted on PR |
| CI/CD | Azure DevOps release pipelines | Reusable GitHub Actions workflows |
| Container runtime | AKS | ECS / Fargate |
| Registry | ACR | ECR (shared account) |
| Secrets | Key Vault + CSI driver | SSM Parameter Store, 1Password |
| Alerting as code | KQL scheduled query rules in Bicep | Splunk alerts in YAML synced via REST |
| Least privilege | Custom Azure roles | Scoped IAM roles, access audits |

## 5-minute prep list

1. Re-read how the AI review prompt is resolved (repo file → default) and the role checklists.
2. Be able to defend **"one directory = one state"** and "no tfvars".
3. Walk through the ECR → ECS deploy end to end, including why the task definition is checked.
4. Explain the credential monitor's **heartbeat and `UNKNOWN`** reasoning.
5. Tell the **SSRF hardening** story in under two minutes.

## Quick-fire Q&A

**How does GitHub authenticate to AWS without keys?**
The job requests an OIDC token (`id-token: write`); AWS STS validates it against the IAM OIDC provider and the role's trust policy (`aud` and `sub` = repo and branch) and returns temporary credentials.

**What do you put in a role trust policy for GitHub?**
`aud = sts.amazonaws.com` and a `sub` condition limited to the repo and branch (or environment) that may deploy.

**Workspaces vs directories in Terraform/OpenTofu?**
Directories give explicit, isolated state and config per environment; workspaces share code and backend and hide the target. This project uses directories.

**How do you verify an ECS deployment really succeeded?**
PRIMARY deployment `COMPLETED` on the new task definition, then the app's health endpoint reports the new version.

**How do you avoid alert storms?**
Dedupe with explicit re-arm, schedule-aware thresholds, and alert catalogs that are reviewed.

**What's "monitoring of monitoring"?**
Heartbeats and `UNKNOWN` states, so a broken checker alerts instead of going quiet.

**How do you patch a fleet safely?**
`--check --diff` first, rolling batches, reboot only when needed, verify health between batches, status page for the window.

**What guardrails do AI agents need?**
Least-privilege tools, human approval for destructive actions, no credential minting, full audit logs, untrusted-input handling.
