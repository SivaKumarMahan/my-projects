# Appendix C — Optional upgrades

[← Appendix B — Adding QA and Prod later](appendix-b-qa-and-prod.md) | [Index](../README.md) | [Appendix D — Command cheat sheet →](appendix-d-cheat-sheet.md)

---

| Upgrade | What you learn |
|---|---|
| **ACR Premium + private endpoint** (`privatelink.azurecr.io`) | Fully private registry; builds must then also run on the self-hosted runner |
| **NAT Gateway** (`outbound_type = "userAssignedNATGateway"`) | Fixed egress IP and more SNAT ports |
| **Azure Firewall + route table** (`outbound_type = "userDefinedRouting"`) | Central egress filtering (FQDN rules for AKS required endpoints), hub-spoke with forced tunnelling |
| **Azure Bastion** in the hub | No public IP on the jumpbox (needs `AzureBastionSubnet`) |
| **Azure Monitor managed Prometheus + Azure Managed Grafana** | Managed alternative to Part 15 (needs `Microsoft.Monitor` and `Microsoft.Dashboard` providers) |
| **Key Vault certificate on App Gateway** (user-assigned identity on App Gateway + `ssl_certificate { key_vault_secret_id }`) | Automatic certificate renewal, no expiry outages |
| **cert-manager + Let's Encrypt** (with a real domain) | Free, auto-renewed public certificates |
| **Argo CD (GitOps)** | Pull-based deploy: the cluster syncs from Git (see your `azure-kubernetes-service/project2-helm-argocd-gitops` repo folder) |
| **Cilium** (`network_data_plane = "cilium"`, overlay) | eBPF dataplane, L7 network policies, Hubble flow logs |
| **Microsoft Defender for Containers / Azure Policy for AKS** | Runtime threat detection; enforce "no privileged pods", "only ACR images" |
| **Commercial scanners** | Snyk, Checkmarx, Veracode, Prisma Cloud, Aqua (commercial Trivy), Mend, GitHub Advanced Security: they replace or extend SonarQube/Trivy/Gitleaks |
| **Terraform modules + Terratest/`terraform test`** | Reusable, tested infrastructure code |

---

[← Appendix B — Adding QA and Prod later](appendix-b-qa-and-prod.md) | [Index](../README.md) | [Appendix D — Command cheat sheet →](appendix-d-cheat-sheet.md)
