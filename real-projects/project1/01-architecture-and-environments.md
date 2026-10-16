# 01 — Architecture and Environments

## Project description

A cloud-native, containerized application that runs warehouse operations on a microservices architecture. It links on-premises warehouse devices (such as Zebra label printers) with a backend hosted in Microsoft Azure. VNet peering, private endpoints and site-to-site VPN keep communication between components secure, scalable and real-time.

### Scope and goals

- Modernize warehouse operations through microservices.
- Enable real-time data flow between on-premises devices and the cloud backend.
- High availability, security and scalability through Azure-native networking.
- Monitoring and observability with Grafana, Prometheus and Azure Monitor.
- Secure access to databases and services with private endpoints and Key Vault.

### Stakeholders

- **Warehouse users** — operate the UI and warehouse devices.
- **DevOps / Infra team** — Azure infrastructure, networking, container registry, deployments.
- **Backend developers** — fulfillment, printing, conveyor, receipt and package-planning services.
- **Monitoring / admin team** — alerts, metrics and logs in Log Analytics, Prometheus and Grafana.

## Architecture components

```
On-prem (Zebra printers, devices)
        │  IPsec site-to-site VPN
        ▼
Virtual Network Gateway (2 public IPs for HA) ── Connection ── Local Network Gateway
        │
   AKS VNet ──── VNet peering ──── VNet in another RG
        │
   ┌────┴──────────────────────────────────────────────┐
   │ AKS cluster (default namespace)                    │
   │  Frontend:  wms-ui-prod                            │
   │  Backend:   aimsplus-wms-production                │
   │             wms-fulfillment-services               │
   │             wms-print-services                     │
   │             wms-conveyor-services                  │
   │             wms-receipt-services                   │
   │             wms-package-planning                   │
   │  CronJob:   ship-fulfillment-sweep                 │
   │  Each: Deployment → ReplicaSet → Pods, Service     │
   └────────────────────────────────────────────────────┘
        │ private endpoint (NIC with private IP in subnet)
        ▼
PostgreSQL Flexible Server (master) ── real-time sync ── Read replica
        (resolved through private DNS zone privatelink.postgres.database.azure.com)

Container Apps Environment (shares existing VNet) → RabbitMQ container app
Container Registry (ACR) → images for all services
Key Vault, NSG, Private DNS Zone
Monitoring: Container Insights → Data Collection Rule/Endpoint → Log Analytics
            Managed Prometheus → Grafana (+ Grafana alerts), Metric Alert Rules
```

### Key features

- **Microservices** — frontend and backend services deployed as containers.
- **Networking** — VNet, VNet peering, private DNS and IPsec VPN.
- **Database** — PostgreSQL Flexible Server with read replicas, reachable only through private endpoints.
- **Monitoring** — Prometheus, Grafana, Log Analytics and metric alert rules.
- **Security** — Key Vault, NSGs and private endpoints to restrict access.

## Environments

| Environment | Resource group | AKS cluster | Region |
|---|---|---|---|
| sandbox | devops-sandbox | sandboxaks | Central US |
| qlab | wms-qlab-01 | qlab-01-aks | Central US |
| performance | performance-aimsplus-westus2 | wmsperformancewestus2 | West US 2 |
| development | new_developments | aimsplusaks | North Central US |
| production (primary, Central) | MC_aimsplusproduction_wmsproductionaks_centralus | wmsproductioncentral | Central US |
| production | aimsplusproduction | aimsplusprodaks | Central US |
| production (West) | aimsplusproduction | wmsproductionaks | West US 2 |
| production-DR | MC_aimsplusproduction_wmsproductionaks_westus2 | — | West US 2 |

All application workloads run in the `default` namespace; the main backend deployment and container are `aimsplus-wms-performance` / `aimsplus-wms-production`.

## Production resource inventory (Central US resource group)

| Type | Name / purpose |
|---|---|
| Virtual network | aks-vnet |
| VNet gateway | aks-vnet-gw with two public IPs (`aks-vnet-public-ip-0`, `-1`) |
| Connections | VNet-to-VNet and site-to-site (Local Network Gateway) |
| NSG | allow-subnets |
| Container registry | ACR for WMS images |
| AKS | wmsproductioncentral |
| Key Vault | wmsproductioncentral |
| Log Analytics | wmsproductioncentral |
| Container Insights | per cluster |
| Data Collection Rule / Endpoint | Container Insights (MSCI-*) and Managed Prometheus (MSProm-*) |
| Private DNS zone | privatelink.postgres.database.azure.com |
| PostgreSQL Flexible Server | wms-production-central |
| Private endpoints + NICs | three private endpoints, each with its own NIC |
| Container App + Environment | wms-rabbitmq |

## AKS node pool layout per environment

| Environment | Pool | Mode | VM size | Scale |
|---|---|---|---|---|
| Sandbox | beast | User | Standard_D2s_v3 | Autoscale, 1 node |
| | app | User | Standard_D2as_v4 | Autoscale, 1 node |
| | system | System | Standard_D2s_v3 | Manual, 1 node |
| Qlab | beast | User | Standard_D8s_v3 | Manual, 1 node |
| | app | User | Standard_D2as_v4 | Manual, 1 node |
| | system | System | Standard_D2s_v3 | Autoscale, 2 nodes |
| Performance | beast | User | Standard_D8s_v3 | Autoscale, 2 nodes |
| | app | User | Standard_D2as_v4 | Autoscale, 2 nodes |
| | system | System | Standard_D2s_v3 | Autoscale, 2 nodes |
| Prod West (wmsproductionaks) | beastpool | User | Standard_D8s_v3 (8 vCPU / 32 GiB) | Manual, 8 nodes |
| | apppool | User | Standard_D2as_v4 | Autoscale 1–8 |
| | agentpool | System | Standard_D2s_v3 | Autoscale 2–4, taint `CriticalAddonsOnly=true:NoSchedule` |
| Prod Central (wmsproductioncentral) | wmsnodes0 / nodes0 | User | Standard_D8s_v3 | Autoscale (raised to 4–8 after incident) |
| | wmsnodes1 / nodes1 | User | Standard_D2as_v4 | Autoscale 1–8 |
| | wmsnodes2 / nodes2 | System | Standard_D2s_v3 | Autoscale 2–4 |

Common node pool settings: Ubuntu Linux, zones 1/2/3, no Spot, 110 max pods per node, no public IP per node, max surge 10%, node drain timeout 30 minutes, labels `type=user` / `type=system`.

## Platform topics covered in a knowledge-transfer session

1. Azure DevOps — repos, pipelines, variable groups, task groups, environments, pipeline variables
2. ARM and Bicep — deployment stacks, custom types
3. Storage account, Log Analytics, Key Vault, Application Insights, PostgreSQL
4. Networking — VNet, NSG, subnet, private endpoints, peering, NIC, gateway, connection, public IP
5. Container Apps, Container Apps Environment, ACR, Container Insights
6. Entra ID, enterprise applications
7. Web Apps — canary, blue-green, deployment slots
8. AKS — networking models, RBAC, upgrades, maintenance
