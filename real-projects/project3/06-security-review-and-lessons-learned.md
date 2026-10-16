# 06 — Security Review and Lessons Learned

A review of the project's DevOps code turned up a set of common anti-patterns. They make a strong interview story: *what I found, why it matters, how I fixed it, and how I'd prevent it next time.*

## Secrets in source control

| Found | Risk | Fix |
|---|---|---|
| SonarQube token hard-coded in a Jenkinsfile | Anyone with repo access can use the token | Jenkins credential + `withCredentials`, rotate the token |
| Database and VM admin passwords in `.bicepparam` files | Plain-text credentials in Git history | `@secure()` param fed from Key Vault (`getSecret()`), pipeline secret, or generated password stored in Key Vault |
| Git personal access token embedded in Terraform module source URLs | Token leaks in code, logs and `.terraform` folders | SSH deploy key or Git credential helper; pin `?ref=<tag>` |
| Server IPs and passwords in a plain-text notes file | Direct access to servers | Password manager / Key Vault, SSH keys only, rotate |
| `.env` files with real values per environment | Secrets on disk and in backups | Commit only `.env.example`; real values in App Service settings / Key Vault references |
| AWS keys passed as Docker build args and set with `ENV` | Credentials stored in image layers | Runtime env vars, instance roles or OIDC; never bake secrets into images |
| Firebase service account key JSON in the repo | Full access to the Firebase project | Store in Key Vault / secret store and load at runtime |
| Service principal password used with `az login` inside pipelines | Long-lived secret in variables | Azure DevOps service connection, ideally workload identity federation |

**Response:** remove from the code, **rotate every exposed credential** (removing from the latest commit isn't enough — Git history and backups still have them), and add prevention.

### Prevention

- **Secret scanning** in pre-commit and CI (gitleaks, detect-secrets, GitHub/GitLab secret scanning).
- **Bicep linter rules** `secure-secrets-in-params`, `outputs-should-not-contain-secrets`.
- `.gitignore` for `.env`, `*.pem`, key files; commit `.env.example` only.
- Secrets only from a vault or the CI credential store.

## Network exposure

| Found | Fix |
|---|---|
| VM NSG rules with source `*` for SSH, vector DB port and VPN port | Source = VPN / hub range only; vector DB port never public; consider Azure Bastion for SSH |
| AWS security group allowing all traffic from `0.0.0.0/0` | Only 80/443 public; everything else from known ranges |
| Storage container with public blob access | Private container + SAS URLs or CDN with private origin, unless the content is genuinely public |
| Key Vault public network access enabled | Private endpoint or selected networks |
| Tomcat directories set to `777`; manager app IP restriction commented out | Least-privilege ownership (`tomcat:tomcat`, `750`); manager restricted or removed |
| `StrictHostKeyChecking=no` on SSH/SCP from pipelines | Known-hosts entry for the target |

## Reliability and correctness issues

| Found | Fix |
|---|---|
| `getopts` string missing `h:` in the Sonar script | Add `h:` and a `usage()` function; `set -euo pipefail` |
| Refactored Sonar pipeline lost the quality gate stage | Re-add `waitForQualityGate()` |
| ChromaDB health check hitting the host port from inside the container | Use the container port (8000) |
| Floating image tags (`chroma:latest`, `terraform:latest`) | Pin versions |
| Terraform `apply`/`destroy` with `-auto-approve` from a dropdown | Saved plan + manual approval |
| CDK validation printed errors but didn't exit | Exit non-zero |
| App settings written with `sites/config` replacing all settings | Merge or manage all settings in one place |
| VNet peering declared twice | Declare once |

## The story to tell

> **Situation:** The project's pipelines and IaC had grown quickly across Azure, AWS, Jenkins and Azure DevOps, with credentials and wide-open rules in several places.
> **Task:** Make the delivery pipeline safe without stopping the team.
> **Action:** I inventoried every place secrets lived, moved them to Jenkins credentials and Key Vault, rotated them, refactored the Sonar pipelines into shared scripts with credentials injected, restricted NSG/SG rules to the VPN range, made the database private-only, and proposed secret scanning plus Bicep linter rules to stop regressions.
> **Result:** No credentials in the repos going forward, the quality gate enforced on all three codebases, and private-only data access. *(Add numbers only if you measured them.)*
