# Part 3 — Networking explained (read before Terraform)

[← Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities](02-bootstrap-state-repo-oidc.md) | [Index](../README.md) | [Part 4 — Terraform stack 1: foundation →](04-terraform-foundation.md)

---

Read this part slowly. Most interview questions about networking are answered by **this** design.

## 3.1 Address plan

| VNet / subnet | CIDR | What lives there | Why this size |
|---|---|---|---|
| **Hub** VNet | 10.0.0.0/16 | shared services | room to grow (Firewall, Bastion, VPN gateway later) |
| ├ snet-jumpbox | 10.0.1.0/24 | jumpbox + GitHub runner VM | |
| **Spoke** VNet | 10.1.0.0/16 | the application | |
| ├ snet-appgw | 10.1.0.0/24 | Application Gateway v2 only | App Gateway needs its **own dedicated** subnet; /24 is the recommended size |
| ├ snet-private-endpoints | 10.1.1.0/24 | private endpoints (PostgreSQL, Key Vault) | one IP per endpoint |
| ├ snet-aks | 10.1.4.0/22 | AKS nodes **and pods** | Azure CNI gives every pod a VNet IP: (nodes + surge) × (max_pods 30 + 1) ≈ 160 IPs during an upgrade; /22 = 1024 |
| AKS service CIDR | 10.100.0.0/16 | ClusterIP Services (virtual) | must **not** overlap any VNet or on-prem range |
| AKS DNS service IP | 10.100.0.10 | CoreDNS Service | must be inside the service CIDR |

**Avoiding overlapping CIDRs (an interview question).** Plan all ranges centrally (an IPAM sheet), and never overlap peered VNets, on-prem ranges, or the AKS service CIDR. Fixing an overlap later means recreating the VNet or cluster, because peering refuses overlapping ranges.

## 3.2 Components and their job

| Component | Layer | Job in this lab |
|---|---|---|
| **VNet** | network | private address space; nothing gets in from the internet unless it has a public IP |
| **Subnet** | network | a slice of the VNet; NSGs and route tables attach here |
| **NSG** | L3/L4 firewall | allow/deny by source, destination, port. Rules are stateful (return traffic is automatic) |
| **VNet peering** | network | connects hub and spoke over the Microsoft backbone; **not transitive** (A-B and B-C does not give A-C) |
| **Private endpoint** | network | a NIC with a **private IP from your subnet** that represents one PaaS resource (PostgreSQL, Key Vault) |
| **Private DNS zone** | DNS | makes the PaaS hostname resolve to that private IP, **but only for VNets linked to the zone** |
| **Application Gateway** | **L7** load balancer | HTTP/HTTPS: host/path routing, TLS termination, cookie affinity, health probes |
| **WAF** (policy on App Gateway) | L7 security | OWASP rules (SQL injection, XSS…) + custom rules; Prevention mode blocks with 403 |
| **Azure Load Balancer** (Standard) | **L4** | TCP/UDP only. AKS uses it for **outbound** (SNAT) traffic and for `Service type: LoadBalancer`. You build an internal one in Part 16 |
| **AGIC** | Kubernetes controller | watches Ingress objects and **writes the App Gateway configuration** through ARM |
| **Ingress** | Kubernetes object | "send traffic for this host/path to this Service" |
| **Ingress controller** | the software that implements Ingress | here AGIC (App Gateway). The alternative is NGINX Ingress behind an internal LB |
| **Azure CNI** | network plugin | pods get **real VNet IPs**, so App Gateway can reach them directly |
| **Calico** | network policy engine | enforces Kubernetes `NetworkPolicy` (pod-level firewall) |
| **CoreDNS** | cluster DNS | resolves Service names; forwards everything else to Azure DNS 168.63.129.16 |

## 3.3 Why the PostgreSQL hostname resolves to a private IP

```text
pod → CoreDNS → Azure DNS (168.63.129.16)
   psql-ttapp-sk01.postgres.database.azure.com
     CNAME psql-ttapp-sk01.privatelink.postgres.database.azure.com
       → private DNS zone "privatelink.postgres.database.azure.com" (linked to spoke + hub)
         A record 10.1.1.5  (created by the private endpoint's DNS zone group)
```

Run the same lookup **from your laptop** and you get the public IP. Because public access is **disabled**, the connection is refused there. That is the whole point: the database is reachable only from inside the VNets. You will see this yourself in Part 8.

**If the DNS zone is not linked to a VNet, that VNet gets the public IP and the connection fails.** This is one of the most common production issues ([Lab 8](17-break-and-fix-lab.md#lab-8--private-dns-zone-link-removed)).

## 3.4 Private AKS: how do we reach the API server?

- The API server gets a **private endpoint** in snet-aks, so there is no public endpoint.
- Its name lives in our **own** private DNS zone `privatelink.<region>.azmk8s.io` (a "custom" private DNS zone). That zone is linked to **both** VNets, so the jumpbox in the hub can resolve it.
- AKS writes the A record into the zone. That's why its control-plane identity needs **Private DNS Zone Contributor** on the zone, and **Network Contributor** on the spoke VNet so it can use the subnet.
- **Who can call it?** Only machines that can route to it (the jumpbox and the runner, over peering) **and** that have an Entra ID role on the cluster (Azure RBAC for Kubernetes).
- **How the pipeline deploys:** a self-hosted runner inside the hub. The alternative, `az aks command invoke`, runs commands through the Azure API without any network path; it is the fallback in Appendix A.

## 3.5 What each NSG allows

| NSG on | Inbound rules (plus Azure defaults) | Why |
|---|---|---|
| snet-appgw | GatewayManager → 65200-65535; Internet → 80, 443 | App Gateway v2 **will not deploy** without the GatewayManager rule |
| snet-aks | App Gateway subnet → 8080 (documentation; the default AllowVnetInBound already allows it) | AGIC sends traffic straight to pod port 8080 |
| snet-private-endpoints | AKS + jumpbox → 5432; VirtualNetwork → 443; **deny everything else from VirtualNetwork** | only these sources can talk to the database |
| snet-jumpbox | **your IP only** → 22 | SSH |

> NSGs only apply to private endpoints if the subnet has `private_endpoint_network_policies = "Enabled"`. Many people miss this.

## 3.6 Outbound traffic (egress)

- **AKS nodes**: `outbound_type = loadBalancer`. The AKS Standard Load Balancer does SNAT through a public IP. Alternatives: a NAT Gateway (more SNAT ports), or `userDefinedRouting` through Azure Firewall (central egress control). See Appendix C.
- **Jumpbox**: has its own public IP.
- **App Gateway**: has its own public IP.

## 3.7 Why App Gateway + AGIC (and not NGINX Ingress)?

| | AGIC + App Gateway (this lab) | NGINX Ingress + internal LB |
|---|---|---|
| Where the L7 proxy runs | outside the cluster (managed PaaS) | inside the cluster (pods) |
| WAF | built in (WAF_v2) | needs ModSecurity or a WAF in front |
| Hops | App Gateway → pod IP directly | LB → NGINX pod → app pod |
| Config | Ingress → AGIC → ARM API (slower to apply: tens of seconds) | instant |
| Needs | Azure CNI (routable pod IPs) | any CNI |
| Cost | App Gateway hourly | only node resources |

Interview line: *"We used App Gateway WAF_v2 with the AGIC add-on, because we needed a managed WAF at the edge and wanted App Gateway to send traffic directly to pod IPs. Inside the cluster, NetworkPolicies allow only the App Gateway subnet to reach the frontend."*

---

[← Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities](02-bootstrap-state-repo-oidc.md) | [Index](../README.md) | [Part 4 — Terraform stack 1: foundation →](04-terraform-foundation.md)
