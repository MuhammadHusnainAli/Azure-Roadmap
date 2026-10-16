# Private Link and Private Endpoints

> Topic: 05 · Networking · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Azure PaaS services are reachable on public endpoints by default. Private Link gives a service a private IP address inside your network instead, so traffic never leaves the Microsoft backbone and the public endpoint can be switched off. It is the foundation of every network-isolated design, and it is non-negotiable for AI workloads that handle internal data: Foundry, AI Search, Storage, Cosmos DB and Key Vault all need it. Most failed private deployments fail on DNS, so this file and the next go together.

## Key concepts

### Terms

| Term | Meaning |
|---|---|
| **Azure Private Link** | The platform capability that delivers a service over a private connection |
| **Private endpoint** | A network interface in your subnet with a private IP, connected to one specific resource (or sub-resource) |
| **Private Link service** | Your own service, behind a Standard Load Balancer, published so consumers can create private endpoints to it |
| **Private link resource** | The target a private endpoint connects to: a storage account, a Foundry resource, a SQL server |
| **Sub-resource (group ID)** | The part of the target being exposed: `blob`, `file`, `account`, `searchService`, `vault`, `sqlServer`, `registry` |

### How a private endpoint works

```text
Your VNet                                      Microsoft backbone
┌──────────────────────────────┐
│ App (10.20.1.10)             │
│      │  DNS: sthr.blob.core.windows.net → 10.20.3.5
│      ▼                       │
│ Private endpoint NIC         │══ Private Link ══►  Storage account "sthr" (blob)
│  10.20.3.5  (snet-pe)        │
└──────────────────────────────┘
```

- The endpoint is a **read-only NIC** with a private IP from your subnet.
- It maps to **exactly one resource and one sub-resource**. A storage account that needs blob and file access privately needs two endpoints. A client holding that IP can reach only that resource, which is what prevents exfiltration to someone else's account.
- Connections are **initiated by the client** toward the service. The service cannot use the endpoint to initiate connections into your network.
- Reachable from the VNet, **peered VNets**, and **on-premises** over VPN or ExpressRoute.
- The private endpoint and the target resource can be in **different regions, subscriptions and tenants**.
- Traffic to the private endpoint is not restricted by the service's IP firewall rules; the service sees a private-link connection.

### Common sub-resources

| Service | Sub-resource(s) |
|---|---|
| Storage | `blob`, `file`, `queue`, `table`, `dfs`, `web` (and `_secondary` variants) |
| Microsoft Foundry / Azure OpenAI / Foundry Tools | `account` |
| Azure AI Search | `searchService` |
| Cosmos DB | `Sql`, `MongoDB`, `Cassandra`, `Gremlin`, `Table` |
| Key Vault | `vault` |
| Azure SQL | `sqlServer` |
| PostgreSQL / MySQL flexible server | `postgresqlServer` / `mysqlServer` |
| Container Registry | `registry` |
| App Service / Functions | `sites` |
| Container Apps environment | `managedEnvironments` |
| API Management | `Gateway` |
| Event Hubs / Service Bus | `namespace` |
| Azure Monitor (through Azure Monitor Private Link Scope) | `azuremonitor` |
| Azure Machine Learning workspace / hub-based projects | `amlworkspace` |

### Approval

| Situation | Flow |
|---|---|
| You have permission on the target resource | **Auto-approved** |
| You know the resource ID but lack permission (cross-team, cross-tenant) | **Manual**: the request is *Pending* until the resource owner approves it |

The resource owner can approve, reject or remove any connection. This gives a clean consumer and provider split.

### DNS: the part that goes wrong

Clients must keep using the service's **normal hostname** (`sthr.blob.core.windows.net`), because TLS certificates and service routing depend on it. Private Link makes that name resolve to the private IP from inside your network.

When a private endpoint exists, the public DNS record becomes a CNAME chain:

```text
sthr.blob.core.windows.net
   CNAME → sthr.privatelink.blob.core.windows.net
               ├─ from the internet:   resolves to the public IP
               └─ inside your network: A record in private DNS zone
                                       "privatelink.blob.core.windows.net" → 10.20.3.5
```

For the private answer to be returned you need:

1. A **private DNS zone** named for the service (`privatelink.blob.core.windows.net`).
2. An **A record** for the resource pointing at the endpoint's IP. A **DNS zone group** on the private endpoint creates and maintains this automatically.
3. The zone **linked** to every VNet whose resolver must return the private answer.
4. Clients using a resolver that consults the zone: Azure DNS (`168.63.129.16`), directly or through a forwarder.

Private DNS zone names for services used in AI workloads:

| Service | Zone |
|---|---|
| Foundry / AI services | `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com`, `privatelink.services.ai.azure.com` |
| Azure AI Search | `privatelink.search.windows.net` |
| Blob / File / Data Lake | `privatelink.blob.core.windows.net`, `privatelink.file.core.windows.net`, `privatelink.dfs.core.windows.net` |
| Cosmos DB (NoSQL) | `privatelink.documents.azure.com` |
| Key Vault | `privatelink.vaultcore.azure.net` |
| Container Registry | `privatelink.azurecr.io` (plus a regional data endpoint record) |
| Azure SQL | `privatelink.database.windows.net` |
| PostgreSQL flexible server | `privatelink.postgres.database.azure.com` |
| App Service / Functions | `privatelink.azurewebsites.net` |
| API Management | `privatelink.azure-api.net` |
| Azure Monitor | Five zones, including `privatelink.monitor.azure.com` and `privatelink.oms.opinsights.azure.com` |

A Foundry resource exposes several hostnames, so its single private endpoint needs records in **all three** zones listed. Missing one gives an environment where some features work and others time out.

Never create a zone named for the public domain (`blob.core.windows.net`). That hijacks resolution for every storage account, including ones without private endpoints.

File 06 covers hybrid resolution, central zones and the fallback behaviour for resources that are not private.

### Public network access

Creating a private endpoint does **not** close the public endpoint. Set `publicNetworkAccess: Disabled` on the resource (or restrict it with firewall rules) as a separate step, and enforce it with Azure Policy.

| Setting | Effect |
|---|---|
| Enabled, no rules | Reachable from anywhere (still requires authentication) |
| Enabled from selected networks | Allowed IPs, VNets (service endpoints), and trusted services only |
| Disabled | Private endpoints only |

Some services offer **trusted services** or **resource instance** exceptions that let specific Azure services reach the resource without a private path. These matter for service-to-service access in AI designs (file 18).

### Network policies on the endpoint subnet

By default NSGs and UDRs do not apply to private endpoints. Enable `privateEndpointNetworkPolicies` on the subnet to:

- filter which sources can reach each endpoint with **NSG** rules; and
- have **UDRs** apply, for example to route endpoint-bound traffic through a firewall.

ASGs can also be attached to private endpoints for rule targeting.

### Managed and shared private endpoints

Some services create private endpoints **on your behalf**, from a network Microsoft manages, to reach your resources:

| Feature | Used by | Purpose |
|---|---|---|
| **Shared private link** | Azure AI Search | Lets indexers and skills reach Storage, Cosmos DB, SQL, Key Vault, Foundry privately |
| **Managed private endpoints** | Data Factory, Synapse, Fabric, Azure Managed Grafana, Stream Analytics | Outbound private access from a managed network |
| **Managed virtual network outbound rules** | Azure Machine Learning; Foundry managed network options | Private outbound from managed compute |

They appear on the target as pending private endpoint connections that you approve. They are how a PaaS service running outside your VNet reaches your locked-down resources.

### Private Link service

To publish **your own** service privately:

1. Put the service behind a **Standard internal Load Balancer** (or use a supported direct-connect configuration).
2. Create a **Private Link service** attached to the load balancer's frontend.
3. Share its **alias** or resource ID. Consumers create a private endpoint to it in their own VNet.
4. Approve connections, optionally restricted by subscription.

The consumer and provider networks **do not need to be connected** and **can have overlapping address spaces**, because the provider sees traffic source-NATed from its own side. Use cases: ISVs offering a service into customers' networks, sharing a service across business units or tenants, and exposing a single service where peering would grant too much. Front Door Premium and Application Gateway can also connect to origins privately through Private Link.

### Limits and cost

- Billed **per endpoint per hour** plus **per GB** processed, in each direction.
- Up to 1,000 private endpoints per VNet and tens of thousands per subscription by default.
- High-scale private endpoint limits can be raised with a subscription feature.

## How it works on Azure

### Deploying a private endpoint properly

Four things, always together:

```bicep
resource pe 'Microsoft.Network/privateEndpoints@2024-05-01' = {
  name: 'pep-aif-hrassist-prod'
  location: location
  properties: {
    subnet: { id: peSubnetId }
    privateLinkServiceConnections: [
      {
        name: 'foundry'
        properties: {
          privateLinkServiceId: foundry.id
          groupIds: ['account']
        }
      }
    ]
  }
}

resource zoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2024-05-01' = {
  parent: pe
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      { name: 'cognitiveservices', properties: { privateDnsZoneId: cognitiveServicesZoneId } }
      { name: 'openai',            properties: { privateDnsZoneId: openAiZoneId } }
      { name: 'services-ai',       properties: { privateDnsZoneId: servicesAiZoneId } }
    ]
  }
}
```

1. The **endpoint** in the PE subnet.
2. The **DNS zone group**, so records are created and removed automatically.
3. The zones **linked** to the right VNets (done once, centrally).
4. **Public network access disabled** on the target.

Azure Verified Modules wrap this in a `privateEndpoints` parameter on each resource module.

### At scale in a landing zone

- Private DNS zones live **once**, in the connectivity subscription, linked to the hub (and to spokes if they resolve directly).
- An Azure Policy with `deployIfNotExists` adds the **DNS zone group** to every new private endpoint automatically, pointing at the central zones. Workload teams create endpoints; DNS just works.
- A policy **denies** creation of `privatelink.*` zones in workload subscriptions to prevent conflicting copies.
- Policies **audit or deny** PaaS resources with public network access enabled.

### Verifying

```bash
# From a machine inside the network
nslookup sthr.blob.core.windows.net          # expect a 10.x address via the privatelink CNAME
curl -sv https://sthr.blob.core.windows.net  2>&1 | grep -E "Connected to|HTTP"

# From Azure
az network private-endpoint show -g <rg> -n <pe> \
  --query "{state:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status, ip:customDnsConfigs[0].ipAddresses[0], fqdn:customDnsConfigs[0].fqdn}"
az network private-endpoint dns-zone-group list -g <rg> --endpoint-name <pe> -o table
az network private-dns record-set a list -g <dns-rg> -z privatelink.blob.core.windows.net -o table
```

If the name resolves to a public IP from inside the network, the problem is DNS: a missing zone link, missing record, or a client using a resolver that does not consult the zone.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Private endpoint or service endpoint | Private endpoint for sensitive data, on-premises access and exfiltration control |
| One PE subnet or several | One per spoke is typical. Separate when different NSG policy is needed |
| Private endpoint in the workload's spoke or centrally | In the consuming workload's spoke. A resource can have several endpoints, one per consuming network |
| Inspect PE traffic through a firewall | Usually not within a workload. Consider it for cross-workload access, accepting routing complexity |
| Public access disabled or restricted | Disabled for internal workloads. For internet-facing services, front them with Application Gateway or Front Door and keep the origin private |
| Cost | Each endpoint has a fixed hourly cost. Dozens of endpoints across many small environments add up; share non-production infrastructure where reasonable |
| Private Link service or peering | Private Link service to expose one service across boundaries |

## Hands-on

Uses the VNet and `snet-pe` from file 01.

```bash
RG=rg-vnet-lab; LOC=swedencentral; VNET=vnet-lab-swc-001
ST=stpelab$RANDOM

# 1. Storage account with public access off
az storage account create -g $RG -n $ST -l $LOC --sku Standard_LRS \
  --public-network-access Disabled --allow-shared-key-access false -o none
ST_ID=$(az storage account show -g $RG -n $ST --query id -o tsv)

# 2. Private DNS zone, linked to the VNet
az network private-dns zone create -g $RG -n privatelink.blob.core.windows.net -o none
az network private-dns link vnet create -g $RG -z privatelink.blob.core.windows.net \
  -n link-$VNET --virtual-network $VNET --registration-enabled false -o none

# 3. Private endpoint with a DNS zone group
az network private-endpoint create -g $RG -n pep-$ST -l $LOC \
  --vnet-name $VNET --subnet snet-pe \
  --private-connection-resource-id $ST_ID --group-id blob --connection-name blob -o none
az network private-endpoint dns-zone-group create -g $RG --endpoint-name pep-$ST -n default \
  --private-dns-zone privatelink.blob.core.windows.net --zone-name blob -o none

# 4. Inspect
az network private-endpoint show -g $RG -n pep-$ST \
  --query "{State:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status, IP:customDnsConfigs[0].ipAddresses[0]}"
az network private-dns record-set a list -g $RG -z privatelink.blob.core.windows.net \
  --query "[].{Name:name, IP:aRecords[0].ipv4Address}" -o table

# 5. From your laptop (outside): the CNAME chain is visible, the address is public, and access is refused
nslookup $ST.blob.core.windows.net
az storage container list --account-name $ST --auth-mode login     # blocked: public access disabled
```

To test from inside, deploy a small VM or a container app into the VNet and repeat the `nslookup`; it should return the `10.20.3.x` address. Then add a second endpoint for the `file` sub-resource and note that it needs its own zone.

Clean up when finished with files 01 to 06: `az group delete -n rg-vnet-lab --yes --no-wait`.

## Security, cost and reliability notes

- **Security:** private endpoints control the network path, not authorisation. Keep Entra authentication and RBAC; disable keys.
- **Security:** always pair with disabled public access, enforced by policy. A private endpoint beside an open public endpoint protects nothing.
- **Security:** review and approve pending connections deliberately. An approved connection from another tenant is access into your resource.
- **Cost:** hourly and per-GB charges per endpoint. Factor into environment cost estimates.
- **Reliability:** private endpoints are highly available within a region. For multi-region resilience create endpoints in each region's network.
- **Reliability:** DNS is the dependency. Monitor resolution, not just connectivity.

## Common mistakes

- Creating the endpoint without DNS, then connecting by IP and failing TLS.
- Missing one of the several zones a service needs (Foundry needs three).
- Zones not linked to the VNet where the client or resolver lives.
- Duplicate `privatelink` zones in workload subscriptions.
- Leaving public network access enabled.
- Forgetting on-premises resolution.
- One endpoint for a storage account while the app also needs `file` or `dfs`.
- Expecting NSGs to filter endpoint traffic without enabling network policies.
- Forgetting the service-to-service paths: the search service or build agent also needs a private route.
- Deleting and recreating a resource and leaving stale A records.

## Check yourself

1. What is a private endpoint, physically?
2. Why does a private endpoint prevent exfiltration to another storage account?
3. Why must clients use the original hostname and not the private IP?
4. What four things make up a correct private endpoint deployment?
5. How many DNS zones does a Foundry resource's private endpoint need?
6. Does creating a private endpoint disable the public endpoint?
7. What is a shared private link in Azure AI Search?
8. When would you publish a Private Link service?

<details>
<summary>Answers</summary>

1. A network interface in your subnet with a private IP, mapped to one resource and sub-resource over Private Link.
2. The endpoint connects only to the one specific resource it was created for; the IP cannot reach any other account.
3. TLS certificates and service-side routing are tied to the hostname; DNS returns the private IP for that same name.
4. The endpoint, a DNS zone group, the private DNS zone linked to the right networks, and public network access disabled on the target.
5. Three: `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com` and `privatelink.services.ai.azure.com`.
6. No. Public network access must be disabled separately.
7. A private endpoint created by the search service from its managed network to your resource, so indexers and skills can reach it privately after you approve the connection.
8. To expose your own service privately to consumers in other VNets, subscriptions or tenants, including where address spaces overlap, without connecting the networks.

</details>

## References

- [What is Azure Private Link?](https://learn.microsoft.com/azure/private-link/private-link-overview)
- [What is a private endpoint?](https://learn.microsoft.com/azure/private-link/private-endpoint-overview)
- [Azure private endpoint private DNS zone values](https://learn.microsoft.com/azure/private-link/private-endpoint-dns)
- [Azure Private Endpoint DNS integration scenarios](https://learn.microsoft.com/azure/private-link/private-endpoint-dns-integration)
- [Private Link and DNS integration at scale](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/private-link-and-dns-integration-at-scale)
- [Manage network policies for private endpoints](https://learn.microsoft.com/azure/private-link/disable-private-endpoint-network-policy)
- [What is Azure Private Link service?](https://learn.microsoft.com/azure/private-link/private-link-service-overview)
- [Configure private link for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [Make outbound connections through a shared private link (Azure AI Search)](https://learn.microsoft.com/azure/search/search-indexer-howto-access-private)
