# Part 19 — Interview questions → where you did it

[← Part 18 — Daily start/stop, cost and teardown](18-start-stop-cost-teardown.md) | [Index](../README.md) | [Appendix A — Blockers and the easy way around →](appendix-a-blockers.md)

---

| Question (from your interview notes) | Where in this lab |
|---|---|
| Explain three-tier architecture / how frontend connects to backend | Part 0, Part 6.3 |
| Which network plugin do you use? Which component gives a pod its IP? | Part 3.2, Part 8.3, Part 8.10 |
| What are network plugins used for? | Part 3.2, Part 16.6 |
| Have you created a private AKS cluster? How? | Part 3.4, Part 8.3, Part 8.10 |
| How does the pipeline authenticate to a private AKS? | Part 2.3, Part 5.3, Part 12 |
| How do you authenticate/connect to AKS? Local accounts? | Part 8.3, Part 8.10 |
| Private endpoint between the AKS backend and the database | Part 3.3, Part 8.4, Lab 7, Lab 8 |
| VNet + subnet + private endpoint: how does the request flow? | Part 0, Part 3.3 |
| Frontend, backend, DB in one VNet or three? | Part 3.1 (subnets + NSGs + NetworkPolicy instead of separate VNets) |
| Overlapping CIDR / address issues | Part 3.1 |
| VNet-to-VNet connectivity | Part 4.5 (peering), Part 5.2 |
| Types of load balancers; which have you used? L4 vs L7 | Part 3.2, Part 3.7, Part 16.7 |
| Health probes in a load balancer | Part 9.6, Part 16.7, Lab 5 |
| What is an Ingress controller? Main configuration of an Ingress | Part 3.2, Part 9.5 |
| Types of Services; how do you expose an app? | Part 16.7, Part 9 |
| Headless service, sidecar, probes | Part 9.6, Part 16.7 |
| Restrict users before requests reach the app (WAF) | Part 13.2 |
| HTTP to HTTPS for frontend and backend | Part 13.1 |
| Certificate expiry without impact | Part 13.1 note, Appendix C |
| How does AKS connect to Key Vault? How do secrets reach the pods? | Part 8.5, Part 9.3, Part 10.4 |
| Service principal vs managed identity; system vs user-assigned | Part 2.3, Part 8.3 |
| Entra Workload ID with a user-assigned identity (detailed steps) | Part 8.5, Part 9.3 |
| If a Secret changes while pods run, what happens? Downtime on rotation? | Part 16.9 |
| Secrets in Docker images / in pipelines | Part 7, Part 2.3 (OIDC), Part 12 |
| Write a sample deployment.yaml | Part 9.3 |
| Multistage Dockerfile, CMD vs ENTRYPOINT, reduce build time | Part 7 |
| SonarQube checks and quality gates | Part 11 |
| How Trivy scans images / what scans in the build stage | Part 7.5, Part 12 |
| DevSecOps for frontend, backend, database | Parts 7, 11, 12, 13 |
| Commercial tools instead of SonarQube/Trivy | Appendix C |
| Write a pipeline for this stack | Part 12.1 |
| Ways a GitHub Actions workflow can run | Part 12.3 |
| Manual approval in the pipeline | Part 2.4 (environment `dev`) |
| Hosted vs self-hosted runners/agents | Part 5.3 |
| Store images in ACR; promote the same image | Part 10.1, Part 12, Appendix B |
| Helm commands for deploy and rollback; deploy a specific version | Part 16.8 |
| Rollback strategy (including database) | Part 16.8 |
| Deployment strategy / zero-downtime | Part 16.1, Part 9.6 |
| HPA, VPA, Cluster Autoscaler; real scenarios; HPA + VPA together | Parts 16.2–16.4 |
| PDB and upgrades; AKS upgrade + challenges; deprecated APIs | Parts 16.5, 16.11 |
| Pod not scheduled / Pending | Lab 4 |
| CrashLoopBackOff | Lab 2 |
| OOMKilled | Lab 3 |
| ImagePullBackOff | Lab 1 |
| 502/504 with all pods Running | Lab 5, Lab 6 |
| App can't connect to PostgreSQL | Lab 7, Lab 8 |
| 10% of traffic failing: investigate | Part 15.7 |
| Integrate Prometheus and Grafana with AKS | Part 15 |
| AKS logs to Azure Monitor; Azure Monitor vs App Insights vs Log Analytics | Part 14 |
| Terraform authentication to Azure | Part 2.3, Part 12.2 |
| Remote state, locking, lock details in plan or apply | Part 2.1, Lab 11 |
| One state file shared by many people | Part 12.5 |
| Modules / multiple environments | Part 0 (two stacks), Appendix B |
| Terraform drift / manual changes | Lab 7, Lab 8 |
| Secure the Terraform state file | Part 2.1 |
| PostgreSQL backup and restore | Part 16.10 |
| Provide RCA | Part 17 (RCA template) |
| Cost optimization in Azure | Part 18.3 |

---

[← Part 18 — Daily start/stop, cost and teardown](18-start-stop-cost-teardown.md) | [Index](../README.md) | [Appendix A — Blockers and the easy way around →](appendix-a-blockers.md)
