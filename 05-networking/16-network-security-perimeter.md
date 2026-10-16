# Network Security Perimeter

> Topic: 05 · Networking · Tracks: E ○ A ● C ○ · Status checked: 2026-10

## Why it matters

Private endpoints secure the path from *your network* to a PaaS service. They do not, by themselves, govern traffic *between* PaaS services, or give you one place to define and audit public access rules across dozens of resources. Network Security Perimeter fills that gap: a logical boundary drawn around PaaS resources, with central access rules and logs. It matters for AI because a RAG system is largely PaaS services calling each other (search to storage, search to model, model to key vault), outside any virtual network you own.

## Key concepts

### The problem it solves

Each PaaS service has its own firewall: IP rules, virtual network rules, trusted-service exceptions, resource instance rules. In a real solution that means:

- **Inconsistency.** Every service expresses rules differently, configured resource by resource.
- **Service-to-service gaps.** Azure AI Search indexing a storage account, or Azure Monitor querying a workspace, comes from Microsoft-managed infrastructure, not your VNet. You end up enabling broad "trusted services" exceptions or shared private links one by one.
- **No exfiltration control on outbound.** Most PaaS firewalls govern inbound only. A compromised service identity could send data to an arbitrary destination.
- **No central visibility** of who accessed what over the public path.

### What a network security perimeter is

A resource (`Microsoft.Network/networkSecurityPerimeters`) that defines a **logical network isolation boundary** for PaaS resources deployed outside your virtual networks.

| Behaviour | Detail |
|---|---|
| **Inside the perimeter** | Associated resources can communicate with each other freely (still subject to authentication and RBAC) |
| **Crossing the perimeter inbound** | Denied by default; allowed only by explicit inbound rules |
| **Crossing the perimeter outbound** | Denied by default; allowed only by explicit outbound rules |
| **Private endpoint traffic** | Always allowed. Private endpoints and perimeters are complementary |
| **Logging** | Access logs for inbound and outbound attempts, allowed and denied |

It is generally available in Azure public cloud and Azure Government regions. Which **services** can be associated varies and is still expanding.

### Components

| Component | Meaning |
|---|---|
| **Perimeter** | The top-level resource |
| **Profile** | A named set of access rules. A perimeter can have several profiles, for groups of resources needing different rules |
| **Access rule** | An inbound or outbound rule within a profile |
| **Resource association** | Links a PaaS resource to a profile, with an **access mode** |
| **Link** (preview / limited) | Connects two perimeters so their resources can communicate |
| **Diagnostic settings** | Send perimeter access logs to Log Analytics, Storage or Event Hubs |

### Access rules

| Direction | Rule types |
|---|---|
| **Inbound** | **IP address ranges** (public CIDRs); **subscriptions** (allows access from resources in the listed subscriptions authenticating with a managed identity); other source types are being added, such as service tags and perimeters |
| **Outbound** | **FQDNs** the associated resources may call |

Resources in the same perimeter do not need rules to reach each other; a managed identity is used to evaluate membership.

### Access modes

Each association has a mode that makes adoption safe:

| Mode | Behaviour |
|---|---|
| **Transition** (previously called *Learning*) | Perimeter rules are evaluated first. If they would deny, the resource's **existing firewall configuration** is evaluated as a fallback. Everything is logged, so you can see what *would* be blocked |
| **Enforced** | Only perimeter rules apply. The resource's own public network settings are ignored; anything not allowed by the perimeter or arriving by private endpoint is denied |

Associate in Transition, read the logs, write rules for the legitimate traffic, then switch to Enforced. It is the same adopt-in-audit-first discipline as Azure Policy and Conditional Access.

The resource's `publicNetworkAccess` property gains a value, **SecuredByPerimeter**, which says "public access is governed by the perimeter". With that set and the association enforced, the resource is locked down to perimeter rules and private endpoints.

### Supported services

Onboarded services include Azure Storage, Key Vault, Azure SQL Database, Cosmos DB, Event Hubs, Service Bus, Azure Monitor (Log Analytics workspaces, Application Insights, alerts, data collection endpoints), and Azure AI Search, with Microsoft Foundry and Foundry Tools (Cognitive Services), and others at varying stages. Some are generally available, some in preview, and capabilities differ per service.

**Check the current list of onboarded private-link resources before designing around it.** For any service not yet supported, use its own firewall, trusted-service settings and private endpoints.

### Limits

Scale limits apply (for example the number of perimeters per subscription, profiles per perimeter, rules per profile and resources per perimeter) and are being enforced as hard limits from 1 November 2026. Check them when designing for a large estate.

## How it works on Azure

### Private endpoints and perimeters together

| Control | Governs | Answers |
|---|---|---|
| **Private endpoint** | Access from your **virtual network** (and on-premises) to a PaaS resource over a private IP | "How do my applications reach this service privately?" |
| **Network Security Perimeter** | **Public-path** access to PaaS resources, and traffic **between** PaaS resources and outward from them | "What else can reach this service, and what can it reach?" |
| **Service firewall** (IP rules, trusted services) | Per-resource public access | Superseded by the perimeter where supported |

They are designed to be used together. A typical isolated design:

- Applications in the VNet reach each PaaS resource through **private endpoints**.
- The PaaS resources are grouped in a **perimeter** so they can call each other without per-pair configuration.
- **No inbound rules** (or only narrowly scoped ones), and **outbound rules** limited to required FQDNs.
- Associations **enforced**; `publicNetworkAccess` set to `SecuredByPerimeter`.

### A RAG workload in a perimeter

```text
┌────────────── Network security perimeter: nsp-hrassist-prod ──────────────┐
│                                                                            │
│   Azure AI Search ──────► Storage (documents)                              │
│        │   └────────────► Foundry (embeddings, skills)                     │
│        │                                                                   │
│   Foundry project ──────► AI Search (retrieval)   Key Vault                │
│                           Cosmos DB (threads)     Log Analytics            │
│                                                                            │
│   Inbound rules:  none (private endpoints only)                            │
│   Outbound rules: none, or specific FQDNs for approved external calls      │
└────────────────────────────────────────────────────────────────────────────┘
        ▲ private endpoints from the workload spoke VNet (always allowed)
```

What this replaces:

| Without a perimeter | With a perimeter |
|---|---|
| A shared private link from search to storage, another to Foundry, each approved manually | Resources in the same perimeter communicate directly |
| "Allow trusted Microsoft services" enabled on storage and key vault | Not needed for in-perimeter traffic |
| Per-resource IP rules for the odd legitimate public caller | One inbound rule on the profile |
| No control on what search or Foundry call outbound | Outbound FQDN rules |
| Access logs scattered per resource | One perimeter access log |

Because support differs per service and feature (for example, which indexer or agent scenarios honour the perimeter), verify each service-to-service path you rely on. Where a path is not yet supported, keep the shared private link or trusted-service configuration for it.

### Adoption steps

1. **Inventory** the PaaS resources in the workload and check each is supported.
2. **Create** the perimeter and a profile.
3. **Associate** resources in **Transition** mode.
4. **Enable diagnostics** to Log Analytics.
5. **Observe** for a representative period, including batch jobs and rarely run processes.
6. **Add rules** for legitimate cross-perimeter traffic revealed by the logs. Question each one.
7. **Switch to Enforced**, resource by resource, starting with non-production.
8. **Govern** with Azure Policy: audit resources not associated with a perimeter; deny public network access values other than disabled or secured-by-perimeter.

### Reading the logs

Log categories distinguish inbound and outbound, allowed and denied, and whether a decision came from perimeter rules or (in Transition mode) the resource's own rules. Example:

```kusto
NSPAccessLogs
| where TimeGenerated > ago(7d)
| summarize hits = count() by Category, MatchedRule, SourceResourceId = tostring(SourceResourceId), DestinationFqdn = tostring(DestinationFqdn)
| order by hits desc
```

Table and column names may differ with the diagnostic configuration. Entries whose category indicates "would be denied in enforced mode" are your to-do list before switching.

### Fronting a perimeter-protected resource

Services that sit **outside** the perimeter but need to reach a resource inside it (for example API Management calling a perimeter-protected back end, or an application on a multi-tenant platform without VNet integration) need an explicit inbound rule, typically by **subscription** with managed identity authentication, or a private endpoint path.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Perimeter or private endpoints | Both. Private endpoints for VNet-to-PaaS; a perimeter for PaaS-to-PaaS, public-path governance and outbound control |
| One perimeter per workload or shared | Per workload (per environment), matching the trust boundary. A shared perimeter lets every member reach every other member |
| Profiles | One profile when all resources share rules; separate profiles when some need different inbound or outbound allowances |
| Inbound by IP or by subscription | Subscription rules with managed identity are more robust than IP ranges, which change |
| Outbound rules | Start with none. Add specific FQDNs with a recorded reason |
| Adopt now or wait | Adopt for supported services in new designs. Where a required service or scenario is in preview, keep existing controls for that path and plan to migrate |
| Relationship to Azure Firewall | The firewall governs egress from your **VNet**. The perimeter governs egress from **PaaS resources**. Different planes; use both |

**Maturity.** This is a newer control than private endpoints, with service coverage and features still growing and some rough edges per service. Treat documentation for each service's perimeter support as authoritative, test the specific flows you depend on, and avoid promising a client capabilities that are still in preview.

## Hands-on

```bash
RG=rg-nsp-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none
az extension add -n nsp 2>/dev/null

# 1. Resources to protect
ST=stnsplab$RANDOM; KV=kv-nsp-lab-$RANDOM
az storage account create -g $RG -n $ST -l $LOC --sku Standard_LRS --allow-shared-key-access false -o none
az keyvault create -g $RG -n $KV -l $LOC --enable-rbac-authorization true -o none
LAW=$(az monitor log-analytics workspace create -g $RG -n log-nsp-lab --query id -o tsv)

# 2. Perimeter and profile
az network perimeter create -g $RG -n nsp-lab -l $LOC -o none
az network perimeter profile create -g $RG --perimeter-name nsp-lab -n default -o none
PROFILE_ID=$(az network perimeter profile show -g $RG --perimeter-name nsp-lab -n default --query id -o tsv)

# 3. Associate in Transition (Learning) mode
for R in $(az storage account show -g $RG -n $ST --query id -o tsv) $(az keyvault show -g $RG -n $KV --query id -o tsv); do
  az network perimeter association create -g $RG --perimeter-name nsp-lab \
    -n assoc-$(basename $R) --access-mode Learning \
    --private-link-resource "{id:$R}" --profile "{id:$PROFILE_ID}" -o none
done
az network perimeter association list -g $RG --perimeter-name nsp-lab \
  --query "[].{Name:name, Mode:accessMode, Resource:privateLinkResource.id}" -o table

# 4. An inbound rule for your current public IP, and an outbound FQDN rule
MYIP=$(curl -s https://api.ipify.org)
az network perimeter profile access-rule create -g $RG --perimeter-name nsp-lab --profile-name default \
  -n allow-my-ip --direction Inbound --address-prefixes "[$MYIP/32]" -o none
az network perimeter profile access-rule create -g $RG --perimeter-name nsp-lab --profile-name default \
  -n allow-learn --direction Outbound --fqdn "['learn.microsoft.com']" -o none

# 5. Diagnostics
NSP_ID=$(az network perimeter show -g $RG -n nsp-lab --query id -o tsv)
az monitor diagnostic-settings create --name to-law --resource $NSP_ID --workspace $LAW \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' -o none

# 6. Switch one association to Enforced and observe the resource's public network access
az network perimeter association update -g $RG --perimeter-name nsp-lab -n assoc-$ST --access-mode Enforced -o none
az storage account show -g $RG -n $ST --query publicNetworkAccess

az group delete -n $RG --yes --no-wait
```

Command and parameter names for this extension have changed between versions; adjust to `az network perimeter --help` if a step fails. In the portal, open the perimeter, review *Associated resources*, the profile's access rules and the diagnostic logs after generating some traffic (for example list blobs from your allowed IP, then from Cloud Shell, which is not allowed).

Design exercise: for the HR assistant (Foundry, AI Search, Storage, Cosmos DB, Key Vault, Log Analytics), list which resources go in the perimeter, the inbound and outbound rules, which service-to-service paths you would verify against current support, and what you keep as a fallback for any unsupported path.

## Security, cost and reliability notes

- **Security:** membership of a perimeter is trust. Do not put resources of different sensitivity or ownership in the same one.
- **Security:** the perimeter controls network reachability. Entra authentication and RBAC still decide what a caller may do. Keep keys disabled.
- **Security:** outbound FQDN rules are the notable addition over classic PaaS firewalls. Use them to bound what a compromised service identity could reach.
- **Security:** leaving associations in Transition mode indefinitely gives logs but no enforcement.
- **Cost:** check current pricing; there is no data-path component to size. Log volume to Log Analytics is the main variable cost.
- **Reliability:** switching to Enforced with an incomplete rule set causes outages in whatever was missed. Observe long enough to capture periodic jobs.

## Common mistakes

- Assuming every PaaS service is supported.
- Treating the perimeter as a replacement for private endpoints.
- Enforcing without a Transition period.
- Never leaving Transition.
- One large perimeter for unrelated workloads.
- Broad inbound IP ranges that recreate an open firewall.
- Forgetting callers outside the perimeter, such as API Management or a pipeline, and breaking them on enforcement.
- Relying on preview behaviour for a production design without a fallback.
- Not sending perimeter logs anywhere.

## Check yourself

1. What gap do network security perimeters fill that private endpoints do not?
2. What is the default behaviour for traffic inside, into and out of a perimeter?
3. What is the difference between Transition and Enforced modes?
4. Is private endpoint traffic affected by perimeter rules?
5. What kinds of inbound and outbound rules can you define?
6. How does a perimeter simplify a search-to-storage-to-model RAG design?
7. Why should each workload usually have its own perimeter?
8. What must you check before designing around this feature?

<details>
<summary>Answers</summary>

1. Governing public-path access and communication between PaaS resources (and outbound from them) centrally, with logging; private endpoints only secure the path from your virtual network.
2. Resources inside communicate freely; inbound and outbound across the boundary are denied unless allowed by explicit rules.
3. In Transition, perimeter rules are evaluated and logged with the resource's own firewall settings as a fallback. In Enforced, only perimeter rules apply.
4. No. Private endpoint traffic is always allowed.
5. Inbound by IP range or by subscription (with managed identity); outbound by FQDN.
6. The services are associated to one perimeter and can reach each other without individual shared private links, trusted-service exceptions or per-resource firewall rules.
7. All members can reach one another, so the perimeter should match a single trust boundary.
8. Whether each service, and the specific service-to-service scenario you need, is currently supported and at what release stage.

</details>

## References

- [What is a network security perimeter?](https://learn.microsoft.com/azure/private-link/network-security-perimeter-concepts)
- [Onboarded private link resources](https://learn.microsoft.com/azure/private-link/network-security-perimeter-concepts#onboarded-private-link-resources)
- [Transition to a network security perimeter](https://learn.microsoft.com/azure/private-link/network-security-perimeter-transition)
- [Diagnostic logs for network security perimeter](https://learn.microsoft.com/azure/private-link/network-security-perimeter-diagnostic-logs)
- [Quickstart: create a network security perimeter](https://learn.microsoft.com/azure/private-link/create-network-security-perimeter-portal)
- [Add an Azure AI Search service to a network security perimeter](https://learn.microsoft.com/azure/search/search-security-network-security-perimeter)
- [Network security perimeter for Azure Storage](https://learn.microsoft.com/azure/storage/common/storage-network-security-perimeter)
- [Configure Azure Monitor with network security perimeter](https://learn.microsoft.com/azure/azure-monitor/fundamentals/network-security-perimeter)
- [Front a perimeter-protected resource with API Management](https://learn.microsoft.com/azure/api-management/using-network-security-perimeter)
