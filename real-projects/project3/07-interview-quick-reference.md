# 07 — Interview Quick Reference

## Two-minute STAR summary

- **Situation:** A new AI-driven news and contracts platform (Node.js API, React admin, React Native app) needed environments on Azure fast — dev, stage and prod — with a private database and AI services.
- **Task:** Build the infrastructure as code, CI with quality gates, and secure network access, with a production approval step.
- **Action:**
  - Exported the first portal-built resources as ARM, then wrote **parameterised Bicep per resource** with naming conventions, `@allowed` environments and per-environment maps.
  - Built a **VNet with computed subnets** (`cidrSubnet()`), App Service VNet integration, hub peering for VPN-only access.
  - Made **MySQL private-only** (private endpoint, private DNS, fixed IP).
  - Deployed **Azure OpenAI models, AI Translator, Storage, Key Vault** and **App Insights wired into the App Service** from Bicep.
  - Set up **Jenkins + SonarQube** for backend, admin UI and mobile app with an enforced **quality gate**, then refactored to shared scripts with credentials injected.
  - Cleaned up **secrets and open network rules** found in the code.
- **Result:** Repeatable environments from code, private data access, enforced code quality on three codebases, and a production approval gate. *(Add measured numbers if you have them.)*

## Talking points by topic

| Topic | Point | File |
|---|---|---|
| Bicep | Per-resource templates, deployment order, `existing` lookups, `cidrSubnet()`, env maps | [02](02-azure-bicep-infrastructure.md) |
| Private networking | Private endpoint + private DNS zone + zone group + A record; delegated subnet for App Service | [02](02-azure-bicep-infrastructure.md) |
| Monitoring | Workspace-based App Insights, diagnostic settings, agent app settings with `slotSetting` | [02](02-azure-bicep-infrastructure.md) |
| CI quality | `withSonarQubeEnv` + `waitForQualityGate` with timeout | [03](03-jenkins-sonarqube-quality-gates.md) |
| Parameters | `readEnvironmentVariable`, `loadYamlContent`, Key Vault `getSecret` | [04](04-azure-devops-pipelines-and-vm-deployments.md) |
| VM deployments | ARM → VM → SSH install → systemd service | [04](04-azure-devops-pipelines-and-vm-deployments.md) |
| Terraform | Workspaces, S3 state, `terraform_remote_state`, dynamic assume-role | [05](05-aws-iac-terraform-cdk-cloudformation.md) |
| CDK / CloudFormation | Config validation, non-root build image, template library, `cfnUpdate` | [05](05-aws-iac-terraform-cdk-cloudformation.md) |
| Security | Secrets inventory, rotation, scanning, least-privilege network rules | [06](06-security-review-and-lessons-learned.md) |

## Quick-fire Q&A

**How does App Service reach a private MySQL server?**
Regional VNet integration through a subnet delegated to `Microsoft.Web/serverFarms`; DNS resolves the server through the private DNS zone to the private endpoint IP.

**Private endpoint vs service endpoint?**
A private endpoint gives the service a private IP in your VNet (traffic never leaves the private network, works from peered/on-prem networks). A service endpoint keeps the public IP but restricts access to selected subnets.

**Why a private DNS zone with a private endpoint?**
So the service's normal hostname resolves to the private IP inside the VNet; without it clients resolve the public address and fail.

**How do you stop a Jenkins build when code quality drops?**
`waitForQualityGate()` after the scan, failing on any status other than `OK`.

**How do you pass secrets to Bicep safely?**
`@secure()` parameters from Key Vault `getSecret()` or pipeline secrets — never plain values in `.bicepparam` files.

**`readEnvironmentVariable` vs a parameters JSON file?**
`readEnvironmentVariable` lets the pipeline supply values from variable groups without generating files; it returns strings, so convert with `bool()` / `int()`.

**What's wrong with putting credentials in Docker build args?**
They're stored in the image history and layers; anyone who pulls the image can read them.

**How do you add an approval before production?**
Jenkins `input` step restricted to an approver group (with a timeout), or Azure DevOps environment approvals and checks.

**Exported ARM templates — use as-is?**
No. They include read-only and generated properties, hard-coded IDs and every default; use them as a reference and write clean, parameterised Bicep.
