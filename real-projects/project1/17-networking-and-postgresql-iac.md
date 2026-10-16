# 17 — Networking and PostgreSQL as Code

## Networking

**Files:** `azure/bicep/stacks/wms/aks-network/main.bicep`, `azure/bicep/modules/network/`, `azure/bicep/modules/dns/private.bicep`

- **VNet and subnets** generated from a CIDR array (`for (address, iter) in subnet_cidrs`).
- **`@batchSize(1)`** on the subnet loop, which avoids the common ARM error from concurrent updates to the same VNet.
- **Bi-directional VNet peering across resource groups:** a `localPeerings` module and a `remotePeerings` module deployed with `scope: resourceGroup(peer.resourceGroup.remote)`, so both sides of the peering live in code.
- **Private DNS zones** linked to the VNet.
- **Delegated subnets:** PostgreSQL uses a delegated subnet plus private DNS when `public_access == 'Disabled'`; the Container Apps environment uses a subnet delegated to `Microsoft.App/environments`.

```bicep
@batchSize(1)
resource subnets 'Microsoft.Network/virtualNetworks/subnets@2023-11-01' = [for (address, iter) in subnet_cidrs: {
  parent: vnet
  name: '${subnetPrefix}-${iter}'
  properties: { addressPrefix: address }
}]

module remotePeering 'peering.bicep' = [for peer in peerings: {
  name: 'remote-${peer.name}'
  scope: resourceGroup(peer.resourceGroup.remote)
  params: { localVnetName: peer.remoteVnet, remoteVnetId: vnet.id }
}]
```

## PostgreSQL Flexible Server

**Files:** `azure/bicep/stacks/wms/postgres-db/main.bicep`, `azure/bicep/modules/database/postgres/`

- **Primary plus optional geo read replica** from one module (`createMode: Replica`, `GeoAsyncReplica`), driven by a params map with `master` and `reader` entries.
- **Virtual endpoints:** a read/write endpoint and a read-only endpoint whose names survive a failover, so apps don't need new connection strings.
- **Admin password from Key Vault at deploy time,** never in code:

```bicep
// params.bicepparam
param administratorLoginPassword = az.getSecret('<sub-id>', '<kv-rg>', '<kv-name>', 'DBPASSWORD')

// main.bicep
@secure()
param administratorLoginPassword string
```

- Entra ID authentication enabled alongside password auth.
- Private networking optional, configurable high availability, storage autogrow, backup retention.

## Interview Q&A

**Why `@batchSize(1)` on subnets?**
Subnets are child resources of one VNet. Parallel PUTs on the same VNet hit `AnotherOperationInProgress`; serialising them avoids it.

**How do you peer VNets that live in different resource groups in one deployment?**
Deploy the remote side as a module with `scope: resourceGroup(<remote rg>)`. The deploying identity needs network contributor rights on both RGs.

**Why virtual endpoints on PostgreSQL?**
After a replica is promoted, the endpoint names point to the new primary, so the application connection strings stay the same.
