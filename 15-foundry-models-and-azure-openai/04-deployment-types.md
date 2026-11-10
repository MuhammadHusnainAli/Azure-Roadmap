# Deployment Types

## Why it matters

When you deploy a model in Foundry, the **deployment type** decides three things clients care about: **where prompts and responses are processed** (data residency), **how you pay** (per token, reserved capacity or discounted batch), and **how predictable latency and throughput are**. It's one of the first questions in any regulated industry ("does our data leave the EU?") and one of the biggest cost levers. Getting it wrong can mean a compliance breach or a large, avoidable bill.

## Key concepts

### The matrix

Deployment types combine a **processing scope** (Global, Data Zone, Regional/Geography) with a **billing model** (Standard, Provisioned, Batch):

| Deployment type | SKU code | Inference processed | Billing | Best for |
|---|---|---|---|---|
| **Global Standard** | `GlobalStandard` | Any Azure region | Pay per token | Default: newest models, lowest price, highest quota |
| **Global Provisioned** | `GlobalProvisionedManaged` | Any Azure region | Reserved PTU (hourly or reservation) | Predictable high throughput, low latency variance |
| **Global Batch** | `GlobalBatch` | Any Azure region | 50% discount vs Global Standard; 24-hour target | Large asynchronous jobs |
| **Data Zone Standard** | `DataZoneStandard` | Within a Microsoft data zone (US, EU or APAC) | Pay per token | Zone-level residency |
| **Data Zone Provisioned** | `DataZoneProvisionedManaged` | Within the data zone | Reserved PTU | Zone residency + predictable throughput |
| **Data Zone Batch** | `DataZoneBatch` | Within the data zone | 50% discount; 24-hour target | Large async jobs with zone residency |
| **Standard** (regional) | `Standard` | Within the Azure geography of the deployment | Pay per token | Geography residency, low–medium volume |
| **Regional Provisioned** | `ProvisionedManaged` | Within the Azure geography | Reserved PTU | Geography residency + throughput |
| **Developer** | `DeveloperTier` | Any Azure region | Pay per token | Evaluating **fine-tuned** models only; 24-hour lifetime, no SLA, no residency guarantee |

**Instant access** (Preview) lets you call some models by name with no deployment at all. It isn't a deployment type.

These types apply to serverless (Microsoft-hosted) deployments. **Managed compute** deployments of open models run on VMs in the region you choose and don't use these SKUs (file 15).

### Data residency: the precise statement

From Microsoft Learn (checked 2026-11):

- **Data stored at rest** (for example uploaded files, fine-tuning data, stored responses) remains in the **designated Azure geography** of the resource for all deployment types.
- **Inferencing data** (prompts and completions) is processed:
  - **Global:** in any Azure region where the model is deployed;
  - **Data Zone:** only within the Microsoft-defined data zone. The **EU** zone follows the **EU Data Boundary** (which can include EFTA countries such as Norway and Switzerland); **US** covers the US; **APAC** covers multiple Asia Pacific regions. Microsoft can add regions to a zone without notice;
  - **Standard and Regional Provisioned:** within the customer-specified Azure **geography**, possibly moving between regions in that geography for operational purposes.

Extended prompt caching stores data on GPU machines within the zone or region boundary for Data Zone and Regional types (file 12).

### Availability order

New models arrive in deployment types in a fixed order:

1. **Global Standard** (at launch)
2. **Global Provisioned**
3. **Data Zone Standard** and **Data Zone Provisioned**
4. **Standard** and **Regional Provisioned** (last, with no guaranteed date, as capacity frees up from retiring models)

Specialised models (audio, image, video) are often available only as Global or Data Zone. So if your policy demands regional processing, you'll often wait longer for new models or can't use them at all.

### Request-level processing tiers

On top of the deployment type, Global Standard and Data Zone Standard support per-request tiers (model-dependent):

| Tier | Effect |
|---|---|
| **Default** | Standard pay-as-you-go |
| **Priority processing** | Higher price for faster, more consistent latency (defined latency targets per model), no long-term commitment |
| **Flex processing** | Lower price for delay-tolerant requests (Global Standard) |

File 12 covers these with batch and caching.

### SLA and latency

| Category | Latency behaviour |
|---|---|
| Standard (all scopes) | Best effort; higher latency variance at high, consistent volume |
| Priority processing | Latency targets per model on pay-as-you-go |
| Provisioned | Dedicated capacity, lower and more consistent latency; latency targets per model |
| Batch | No real-time SLA; target completion within 24 hours |
| Developer | No SLA |

## How it works on Azure

### Choose and create

```bash
# Data Zone Standard in an EU region keeps inference within the EU Data Boundary
az cognitiveservices account deployment create -g rg-foundry-eu -n fdy-contoso-weu \
  --deployment-name gpt-5-mini-dz --model-format OpenAI --model-name gpt-5-mini --model-version 2025-08-07 \
  --sku-name DataZoneStandard --sku-capacity 100
```

Remember that the **resource's region** determines the data zone or geography: an EU data zone deployment requires a resource in an EU region.

### Enforce allowed deployment types with Azure Policy

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "equals": "Microsoft.CognitiveServices/accounts/deployments" },
        { "field": "Microsoft.CognitiveServices/accounts/deployments/sku.name", "in": [ "GlobalStandard", "GlobalProvisionedManaged", "GlobalBatch" ] }
      ]
    },
    "then": { "effect": "deny" }
  }
}
```

This example denies Global types in subscriptions where EU residency is mandatory.

### Decision flow

```text
Must inference stay in a specific geography?   ── yes → Standard / Regional Provisioned (check model availability)
        │ no
Must it stay within the US/EU/APAC data zone? ── yes → Data Zone Standard / Data Zone Provisioned / Data Zone Batch
        │ no
Async, not time-sensitive, large volume?      ── yes → Global Batch (or Data Zone Batch)
        │ no
Steady high volume or strict latency?         ── yes → Global Provisioned (or Priority processing)
        │ no
Default                                        → Global Standard
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Residency vs capability | Stricter scopes reduce model choice and delay access to new models; agree the real legal requirement (often data zone, not single region) with compliance |
| Global by default | Microsoft recommends starting with Global Standard unless residency, reserved throughput or batch is needed |
| Mixing types | Use different types per workload in the same resource: Data Zone for customer data, Global Batch for synthetic test data generation, Provisioned for the main chat path |
| Resilience | Global and Data Zone types route dynamically, but an outage in the resource's primary region still affects traffic routed through it; deploy in two regions for high availability (file 13) |
| Government clouds | Global Standard isn't available in Azure Government; model lists differ |

## Hands-on

1. Deploy the same model as Global Standard and Data Zone Standard in an EU resource; compare default quota and latency.
2. Create a Global Batch deployment and submit a small batch file (file 12).
3. Assign the deny policy above to a test subscription and confirm a Global deployment is blocked.
4. Read your organisation's data residency policy and map each AI workload to an allowed deployment type.

## Security, cost and reliability notes

- **Compliance:** document the deployment type per workload in your AI system inventory. Auditors will ask.
- **Cost:** Global Standard is usually cheapest per token; Data Zone and Regional cost more. Batch saves 50% for suitable work.
- **Reliability:** Standard types can see latency variance at scale; Provisioned or Priority provide predictability.
- **Governance:** enforce allowed deployment types with Azure Policy and review new deployments in the Control Plane.

## Common mistakes

- Promising "data never leaves the region" on a Global deployment.
- Assuming data at rest moves with Global processing (it stays in the geography).
- Requiring regional processing without checking that the needed models exist in Standard or Regional Provisioned.
- Using Developer deployments in production.
- Forgetting that a Data Zone deployment's zone comes from the resource's region.

## Check yourself

1. Name the SKU codes for Global Standard, Data Zone Provisioned and Regional Provisioned.
2. Where are prompts processed for Global, Data Zone and Standard deployments?
3. Where is data at rest stored?
4. In what order do new models become available across deployment types?
5. What's the Developer deployment type for?
6. What discount and turnaround does Batch offer?

<details>
<summary>Answers</summary>

1. `GlobalStandard`, `DataZoneProvisionedManaged`, `ProvisionedManaged`.
2. Global: any Azure region; Data Zone: within the US, EU (EU Data Boundary) or APAC zone; Standard: within the Azure geography of the deployment.
3. In the designated Azure geography of the resource, for all deployment types.
4. Global Standard, then Global Provisioned, then Data Zone Standard and Provisioned, then Standard and Regional Provisioned.
5. Evaluating fine-tuned models cheaply, with a 24-hour lifetime and no SLA or residency guarantee.
6. 50% less than Global Standard, with a 24-hour target turnaround and separate enqueued-token quota.

</details>

## References

- [Deployment types for Foundry Models](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)
- [Region availability for Foundry Models sold by Azure](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability)
- [Azure data residency](https://azure.microsoft.com/explore/global-infrastructure/data-residency/)
- [EU Data Boundary](https://learn.microsoft.com/privacy/eudb/eu-data-boundary-learn)
- [Priority processing](https://learn.microsoft.com/azure/foundry/openai/concepts/priority-processing)
- [Foundry Models lifecycle: model launch and availability](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirements#model-launch-and-availability)
