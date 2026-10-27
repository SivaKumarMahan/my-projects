# Hands-on: Three-tier application on private AKS (end to end)

This guide is a lab you build yourself, step by step. At the end you will have **done** everything you currently explain only in theory in interviews:

- **Terraform**: remote state, two stacks, every Azure resource
- **Networking**: hub-spoke VNets, subnets, NSGs, peering, private endpoints, private DNS zones, Application Gateway + WAF, load balancers, Ingress
- **Private AKS**: Entra ID login, Azure CNI, Calico network policies, probes, HPA, VPA, Cluster Autoscaler, PDB, upgrades
- **Application**: React frontend, Spring Boot backend, PostgreSQL, multistage Dockerfiles, Helm chart
- **CI/CD**: GitHub Actions with OIDC (no passwords), SonarQube Cloud, Trivy, Gitleaks, ACR, self-hosted runner
- **Security**: Key Vault + Secrets Store CSI driver + Workload Identity, WAF, non-root containers
- **Monitoring**: Azure Monitor (Container Insights, Application Insights, Log Analytics, alerts) and Prometheus + Grafana
- **Operations**: rollback, zero-downtime deploys, secret rotation, AKS upgrade, and a **break-and-fix lab** for the troubleshooting questions

The topics come from the questions in your `interview-questions` repository (the `real-interview-questions` and `repetitive-questions` folders). [Part 19](steps/19-interview-questions-map.md) maps each question to the step where you did it.

> **How this guide was tested.** All code in this guide was checked before it was written down. Both Terraform stacks pass `terraform validate` with azurerm 5.11. The Spring Boot backend builds and its tests pass. The frontend and backend images were built and run together with PostgreSQL. Trivy, Gitleaks, Helm lint, kubeconform and actionlint all pass. Nothing was deployed to Azure, so the Azure steps can still fail because of your subscription's policies, quotas or region. The [blockers appendix](steps/appendix-a-blockers.md) lists the likely problems and the easy way around each one.

---

## How to use this guide

Work through the files in `steps/` in order. Each one ends with a link to the next.
Parts 0–18 are the build. Part 19 maps interview questions to the steps. The appendices are for reference when you need them.

| Step | File |
|---|---|
| Part 0 — What you will build | [steps/00-what-you-will-build.md](steps/00-what-you-will-build.md) |
| Part 1 — Prerequisites and permission checks | [steps/01-prerequisites-and-permissions.md](steps/01-prerequisites-and-permissions.md) |
| Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities | [steps/02-bootstrap-state-repo-oidc.md](steps/02-bootstrap-state-repo-oidc.md) |
| Part 3 — Networking explained (read before Terraform) | [steps/03-networking-explained.md](steps/03-networking-explained.md) |
| Part 4 — Terraform stack 1: foundation | [steps/04-terraform-foundation.md](steps/04-terraform-foundation.md) |
| Part 5 — Jumpbox and GitHub self-hosted runner | [steps/05-jumpbox-and-runner.md](steps/05-jumpbox-and-runner.md) |
| Part 6 — The application code | [steps/06-application-code.md](steps/06-application-code.md) |
| Part 7 — Multistage Dockerfiles and local scans | [steps/07-dockerfiles-and-local-scans.md](steps/07-dockerfiles-and-local-scans.md) |
| Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL) | [steps/08-terraform-platform.md](steps/08-terraform-platform.md) |
| Part 9 — The Helm chart | [steps/09-helm-chart.md](steps/09-helm-chart.md) |
| Part 10 — First deployment by hand | [steps/10-first-deployment.md](steps/10-first-deployment.md) |
| Part 11 — SonarQube Cloud | [steps/11-sonarqube-cloud.md](steps/11-sonarqube-cloud.md) |
| Part 12 — GitHub Actions CI/CD | [steps/12-github-actions-cicd.md](steps/12-github-actions-cicd.md) |
| Part 13 — HTTPS and WAF | [steps/13-https-and-waf.md](steps/13-https-and-waf.md) |
| Part 14 — Monitoring with Azure Monitor | [steps/14-azure-monitor.md](steps/14-azure-monitor.md) |
| Part 15 — Monitoring with Prometheus and Grafana | [steps/15-prometheus-and-grafana.md](steps/15-prometheus-and-grafana.md) |
| Part 16 — Kubernetes exercises | [steps/16-kubernetes-exercises.md](steps/16-kubernetes-exercises.md) |
| Part 17 — Break-and-fix lab | [steps/17-break-and-fix-lab.md](steps/17-break-and-fix-lab.md) |
| Part 18 — Daily start/stop, cost and teardown | [steps/18-start-stop-cost-teardown.md](steps/18-start-stop-cost-teardown.md) |
| Part 19 — Interview questions → where you did it | [steps/19-interview-questions-map.md](steps/19-interview-questions-map.md) |
| Appendix A — Blockers and the easy way around | [steps/appendix-a-blockers.md](steps/appendix-a-blockers.md) |
| Appendix B — Adding QA and Prod later | [steps/appendix-b-qa-and-prod.md](steps/appendix-b-qa-and-prod.md) |
| Appendix C — Optional upgrades | [steps/appendix-c-optional-upgrades.md](steps/appendix-c-optional-upgrades.md) |
| Appendix D — Command cheat sheet | [steps/appendix-d-cheat-sheet.md](steps/appendix-d-cheat-sheet.md) |
