# 12 — RabbitMQ Migration, Database Networking and Global Load Balancing

## Part A — Migrate RabbitMQ from Container Apps to AKS

### Current state

- RabbitMQ runs as an Azure Container App deployed with Bicep.
- Publicly reachable; services talk to it over public endpoints.

### Target state

- RabbitMQ as a Kubernetes workload inside AKS, in its own `rabbitmq` namespace.
- **ClusterIP** Service for internal traffic; Ingress only for the management UI if needed.
- RBAC, NetworkPolicies and TLS to lock down traffic.
- **PersistentVolumes / PVCs** for message durability.
- Horizontal pod autoscaling for load changes.

### Steps

1. Create the `rabbitmq` namespace (manifest / Helm).
2. Deployment with persistent storage, ClusterIP Service, liveness and readiness probes, authentication enabled for RabbitMQ users.
3. Export and migrate existing queues, exchanges and bindings if needed; verify after migration.
4. Test inter-service communication inside the cluster; restrict any external access with ACLs; load test.

RabbitMQ was packaged as a Helm chart (`values/wms/rabbitmq/values.yaml`) with its own release pipeline, using a managed-identity service connection backed by an AKS-specific custom role.

## Part B — Bring PostgreSQL into the AKS network

### Current state

- PostgreSQL Flexible Server in a **separate VNet**.
- Publicly accessible with network ACL restrictions.
- The application reaches it through a private link / endpoint.

### Target state

- PostgreSQL and AKS in the **same VNet** (dedicated DB subnet).
- No public exposure; private IPs only.
- Bastion host for admin access.
- Private Link, NSGs and firewall rules controlling access.

### Steps

1. **Bicep networking** — update AKS and PostgreSQL templates to share the VNet; create the new server in a dedicated subnet; remove public IP dependency; make sure private DNS resolves the private endpoint.
2. **Migrate data** — `pg_dump` from the old server and restore into the new one.
3. **Private access** — AKS nodes reach the DB on private IPs; firewall allows only the Kubernetes subnet; RBAC for DB users; update the DB stack's network settings.
4. **App connection strings** — point to private endpoints; enforce SSL/TLS.
5. **Validate** — connectivity from inside the cluster, security rules, and latency benchmarks before vs after.

## Part C — Global load balancer across regions

### Design evaluated (Azure-native)

```
                 Clients
                    │
        Global Load Balancer (public frontend IP)
           ┌────────┴────────┐
   Backend pool 1       Backend pool 2
           │                 │
 Regional LB – East US   Regional LB – West US
   ├─ pool 1: app          ├─ pool 1: app
   └─ pool 2: redundancy   └─ pool 2: redundancy
```

- **Global LB** — single public entry point; routes to regional LBs by latency, geo-proximity or availability.
- **Regional LBs** — receive traffic from the global LB and spread it across local backends, with a second pool for redundancy.
- Goals: high availability, traffic distribution and fault tolerance across regions.

### Decision

Global load balancing was implemented in **Cloudflare Load Balancing** in front of the regional endpoints instead of deploying a cross-region load balancer plus multiple regional load balancers in Azure. This kept Azure costs and complexity down, and Cloudflare already provided DNS, TLS and CDN for the application. It pairs with the multi-region release pipeline (west2 primary / central failover) in [10](10-application-build-and-release-flow.md).
