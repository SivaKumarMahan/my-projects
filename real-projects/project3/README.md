# Real Projects — Project 3: AI News and Contracts Platform on Azure (NGL)

DevOps work for a multi-tenant platform that collects supply-chain news, translates and summarises it with Azure OpenAI, matches similar articles with vector search, and pushes notifications to a mobile app. It also manages companies, contracts, pricing and locations for client users.

The project combines **Azure infrastructure in Bicep** (with exported ARM templates as reference), **Jenkins pipelines with SonarQube quality gates**, and a reusable **IaC toolkit** (Azure DevOps pipelines, Terraform, AWS CDK and CloudFormation) used to bootstrap environments.

## Project at a glance

| Area | What was used |
|---|---|
| Backend | Node.js 20 + TypeScript, Express, Sequelize (MySQL), Swagger, node-cron jobs, Winston logging to Application Insights |
| AI | Azure OpenAI (`gpt-35-turbo-16k` for summaries, `text-embedding-3-large` for embeddings), Azure AI Translator, LangChain, ChromaDB / Pinecone vector stores |
| Admin UI | React + Vite + TypeScript, React Bootstrap — hosted on Azure Static Web Apps |
| Mobile app | React Native (Android / iOS), Firebase (messaging, analytics, crashlytics, app distribution) |
| Azure | App Service (Linux), Static Web Apps, MySQL Flexible Server, Private Endpoints + Private DNS, VNet + peering, Key Vault, Storage (blob), Log Analytics, Application Insights, Linux VM (ChromaDB) |
| IaC | Bicep (per-resource templates + `.bicepparam`), ARM templates, Terraform, AWS CDK (Python), CloudFormation |
| CI/CD | Jenkins (declarative pipelines), SonarQube quality gates, Azure DevOps YAML pipelines |
| Environments | `dev`, `devops`, `stage`, `prod` |
| Network access | VPN-only access to non-public services |

## Documents

| # | File | Topic |
|---|---|---|
| 01 | [01-application-architecture-and-environments.md](01-application-architecture-and-environments.md) | Components, data flow, environments, data model, configuration |
| 02 | [02-azure-bicep-infrastructure.md](02-azure-bicep-infrastructure.md) | Per-resource Bicep templates, deployment order, private MySQL, VNet design, App Insights wiring |
| 03 | [03-jenkins-sonarqube-quality-gates.md](03-jenkins-sonarqube-quality-gates.md) | Jenkins pipelines, SonarQube scanning, quality gate, script refactor |
| 04 | [04-azure-devops-pipelines-and-vm-deployments.md](04-azure-devops-pipelines-and-vm-deployments.md) | ARM/Bicep pipelines, VM + Tomcat deployment, static website, AKS from Bicep, parameter techniques |
| 05 | [05-aws-iac-terraform-cdk-cloudformation.md](05-aws-iac-terraform-cdk-cloudformation.md) | Terraform in Docker via Jenkins, remote state, VPC peering, CDK with validation, CloudFormation library |
| 06 | [06-security-review-and-lessons-learned.md](06-security-review-and-lessons-learned.md) | Secrets and network issues found in the code, and the fixes |
| 07 | [07-interview-quick-reference.md](07-interview-quick-reference.md) | STAR summary, talking points, quick-fire Q&A |
