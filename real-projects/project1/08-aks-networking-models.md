# 08 — AKS Networking Models: Kubenet vs Azure CNI

The existing clusters run **kubenet**. This comparison was written to decide whether to move to Azure CNI, especially for direct pod-to-database connectivity over private endpoints.

## Overview

- **Kubenet** — default lightweight plugin. Linux bridge networking plus Azure route tables. Pods get IPs from a CIDR **outside** the VNet and are NAT'ed through the node NIC.
- **Azure CNI** — full VNet integration. Each pod gets a routable VNet IP (or an overlay IP, depending on variant) and talks directly to Azure resources.

## Comparison

| Feature | Kubenet | Azure CNI |
|---|---|---|
| Networking | Basic | Advanced |
| Access to Azure services | Indirect, via NAT on node NIC | Direct to DBs, storage, other VNet resources, peered VNets, on-prem |
| Pod IP | Private, non-routable CIDR outside VNet | From VNet subnet, or overlay |
| Node NIC | One NIC, NAT for pods | One NIC, IP forwarding; by default 30 extra IPs pre-allocated per node |
| Routing | NAT-based | Direct (or NAT for overlay) |
| IP planning | Small IP space | Consumes VNet IPs (or overlay / dynamic) |
| VNet integration | Limited, adds latency | Full, incl. peered VNets |
| Performance | Lower; NAT latency for network-heavy apps | Good; direct pod-to-pod / pod-to-VNet |
| Network policies | Basic Kubernetes policies | Advanced (Cilium, Azure) |
| Scalability | Easy, low IP use | Highly scalable with overlay / dynamic |
| Azure integration | Limited | Private Link, service endpoints, Azure network policies |
| Use case | Lightweight, cost-conscious | Enterprise, large, production-grade, hybrid |

## When to choose

**Kubenet**
- Low IP consumption (large clusters in small VNets)
- Cheaper and simpler routing
- Simple setup for dev/test
- Outbound NAT is fine for external API calls
- Small/medium workloads without direct Azure service integration

**Azure CNI**
- Direct VNet connectivity to Azure services
- No NAT overhead → lower latency
- Stronger network policies for security/compliance
- Large-cluster scaling

## Azure CNI variants

| Variant | Best for |
|---|---|
| Traditional (VNet IPs) | Small–medium clusters with plenty of VNet IP space; lowest latency |
| Dynamic IP allocation | Large clusters needing efficient IP management |
| Overlay | Large clusters with limited VNet IP space; mostly internal pod traffic |
| Powered by Cilium | Advanced security, encryption, observability (eBPF) |
