# 03 — Bicep Testing and Validation

Before moving production infrastructure to Bicep, the templates were tested in a sandbox to prove that redeployments are idempotent, that AKS version changes behave predictably, and that failure modes are understood.

## Core requirements

- **TestCase 1.0** — deploy multiple resources (VNet, AKS, Storage Account) from **one** Bicep template. All properties must match the template.
- **TestCase 2.0** — redeploy the same resources from **individual** templates. Nothing should change in the portal; any change is recorded and investigated.
- No hard-coded values in templates — everything comes from parameter files, and the same values are used across all test cases.
- Clean-up checked with `az deployment sub delete` for subscription-scope deployments.

## Test matrix

| Scenario | Resources | Expected result |
|---|---|---|
| Merge several templates into one and deploy | VNet, AKS, Storage | No changes in portal |
| Deploy individual templates without changes | VNet, AKS, Storage | No changes in portal |
| Change one resource in a multi-resource template | VNet, App Service, Storage | Only that resource updates |
| Change a resource in the portal, then redeploy | VM, LB, App Service | Portal drift is overwritten by Bicep |
| Change in portal **and** in template, then deploy | VM, App Service, Key Vault | Observe final state |
| Omit optional parameters | VNet, AKS, Storage | Defaults applied |
| Remove required parameters | Storage, App Service, Key Vault | Deployment fails |
| Invalid values | VNet, AKS, Storage | Deployment fails |
| Resources with dependencies | VNet/Subnet, AKS, Storage blob | Created in correct order |
| Missing dependency definitions | VM, App Service Plan, DB | Deployment fails |
| Sensitive values in template | Key Vault, App Service, DB | Not exposed in logs or portal |
| Add a resource | Key Vault, App Service, DB | New resource only, nothing else touched |
| Remove a resource | VNet, AKS, Storage | Resource deleted, others unaffected |
| Change a property | VM, App Service, DB | Only that property changes |
| Intentional error | VNet, AKS, Storage | Check rollback behaviour |
| One resource fails in a multi-resource deploy | VNet, VM, DB | Only failing resource affected |
| Parent template calling child modules | VNet, AKS, Storage | All module resources deployed |
| Dependencies between modules | VNet, Key Vault, DB | Correct order |
| Role assignments in template | Key Vault, AKS, Storage | Correct roles assigned |
| Different parameter files for Dev/QA/Prod | VM, App Service, DB | Environment-specific config |
| Exceed service limits | VM, Storage, DB | Quota error |
| Deploy to different regions | VNet, App Service, DB | Success in all regions |
| Scale a resource | VM, AKS, App Service | Scales without side effects |
| Complete mode | VNet, AKS, Storage | Only template resources remain |
| Nested loops and conditions | VM, Storage, VNet | Correct resources created |
| Same module, different parents/params | VNet, AKS, Storage | Consistent behaviour |
| Cancel mid-deployment | VM, AKS, App Service | Redeploy completes cleanly |
| Resource lock | VNet, Storage, DB | Clear failure on locked resource |
| Deny policy | VNet, Key Vault, Storage | Policy violation error |
| Deploy / update / delete repeatedly | VNet, App Service, Storage | Consistent results |

## AKS version changes through Bicep

| Case | Control plane | Node pool |
|---|---|---|
| 1.1 | 1.29.0 | 1.29.0 |
| 2.1 | 1.30.0 | 1.29.0 |
| 2.2 | 1.30.0 | 1.30.0 |
| 2.3 | 1.31.1 | 1.30.0 |
| 2.4 | 1.30.0 | 1.31.1 |
| 2.5 | 1.31.1 | 1.31.1 |
| 2.6 | 1.31.1 | 1.30.0 (downgrade) |

**What was monitored:** pod and node availability, how pods/nodes move, restarts, new nodes created and old ones removed.

### Results

- **1.1 (1.29 / 1.29)** — all pods and nodes reachable. An NGINX deployment exposed as `LoadBalancer` caused AKS to automatically create a new public IP on the Azure Load Balancer, a load-balancing rule and health probe, and an NSG inbound rule (`k8s-azure-lb_allow_IPv4_…`). The service was reachable externally.
- **2.1 (control plane → 1.30)** — nodes unchanged (version, IP, image, runtime). Only system pods (cloud-node-manager, CSI disk/file, konnectivity-agent, kube-proxy, metrics-server) were rescheduled. No application restarts; LB still working.
- **2.2 (node pool → 1.30)** — nodes recreated with the same name on the new version; all pods recreated (new names). No downtime observed.
- **2.3 (control plane → 1.31.1)** — same pattern as 2.1; only system pods rescheduled.
- **2.4 (control plane 1.31.1 → 1.30.0)** — **failed**. Kubernetes does not allow downgrades; the only option is a new cluster.
- **2.5 (node pool → 1.31.1)** — nodes recreated and all pods rescheduled; konnectivity-agent pods recycled. One node showed timeouts, consistent with VM maintenance during the rollout.
- **2.6 (node pool 1.31.1 → 1.30.0)** — **failed** for the same reason: no node pool downgrades.

### Conclusion

Redeploying unchanged templates (single or split) produced **no configuration changes** in Azure — the templates are idempotent. A control-plane-only upgrade leaves nodes in place and only cycles system pods; a node pool upgrade recreates nodes and all pods. Downgrades are blocked at both levels.

## Lab walkthrough

The full lab — templates, parameter files, deployment commands, resources Azure created, monitoring commands, functional tests and clean-up — is in [20-bicep-testcases-lab-walkthrough.md](20-bicep-testcases-lab-walkthrough.md).
