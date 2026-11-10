# Quotas, Regions and Pricing

## Why it matters

The first production surprise for many AI teams is that the model they need isn't available in their region, their quota is too small for launch-day traffic, or the bill is far higher than the proof of concept suggested. Quotas, regional availability and pricing meters shape architecture: which region, which deployment type, whether to buy provisioned throughput, how to attribute cost to teams. Consultants need this to estimate; architects need it to design.

## Key concepts

### Regional availability

- Models, deployment types and features (agents, evaluations, fine-tuning, hosted agents) **vary by region**. New models often launch in a few regions or as **Global** deployments first.
- **Global** deployments can run in any Azure region where the model is hosted (routing is Azure-managed), giving the widest availability and highest default quota. **Data Zone** keeps processing within the US or EU zone. **Standard (regional)** processes in the deployment's region.
- Data **at rest** stays in the resource's geography regardless of deployment type. Deployment type governs where **inferencing** happens (topic 15 file 04).
- Always check the region availability tables before choosing a region, and re-check before production.

### Quota model (checked 2026-11)

| Concept | Detail |
|---|---|
| **Units** | Tokens per minute (**TPM**) and requests per minute (**RPM**), assigned per model and deployment type. RPM is tied to TPM by a ratio that **varies by model and deployment type**. In the 2026 tier tables, for example, gpt-4.1 Global Standard allows 1 RPM per 1,000 TPM and gpt-5 allows 10 RPM per 1,000 TPM. The older "6 RPM per 1,000 TPM" rule of thumb no longer holds, so check the model's quota table |
| **Allocation** | You assign part of your available quota to each deployment (deployment "capacity"). Unassigned quota stays available for other deployments |
| **Subscription-level pools** | Since May 2026 Foundry is moving quota to the **subscription level**: Global Standard deployments of the same model and version share **one pool across all regions** in the subscription; Data Zone Standard shares one pool per data zone. This started with some models and is rolling out to all |
| **Quota tiers** | Seven tiers: **Free (Tier 0)** and **Tiers 1–6**. Initial tier depends on usage and relationship (EA and MCA-E customers start higher). Subscriptions **auto-upgrade** to higher tiers as consumption grows (opt-out available); payment history matters. Free tier can't request increases |
| **Quota requests** | Eligible subscriptions submit the quota request form; approval adds quota without changing tier |
| **Provisioned throughput (PTU) quota** | Separate quota in PTUs per deployment type; capacity is also subject to regional availability (topic 15 file 05) |
| **Other limits** | For example 30 Azure OpenAI resources per region per subscription, 32 standard deployments per resource, 2,048 inputs per embeddings call, image and video request limits |

Quota limits change frequently. Treat the quota page as the source of truth and record the date you checked.

### Rate limiting behaviour

- Exceeding TPM or RPM returns **HTTP 429** with a `retry-after` header (and `x-ratelimit-*` headers).
- Limits are evaluated over short windows, so bursts can hit 429 even when the per-minute average is fine.
- `max_tokens` / `max_output_tokens` affects TPM accounting estimates, so set realistic values.
- Topic 15 file 13 covers retries, backoff and multi-region resilience.

### Pricing meters

| Meter | Applies to |
|---|---|
| **Input tokens, cached input tokens, output tokens** | Standard (pay-as-you-go) deployments: Global, Data Zone and Regional rates differ, with Global usually cheapest |
| **Short-context vs long-context rates** | Some newer models price requests by input length category (for example GPT-5.6 and GPT-6 families) |
| **Batch** | Global/Data Zone Batch, at a discount to standard (topic 15 file 12) |
| **Priority processing / Flex** | Premium or discounted tiers for latency vs cost, where available |
| **Provisioned throughput units (PTU)** | Hourly per PTU, with monthly or yearly **reservations** for discounts |
| **Images, audio, video** | Per image, per audio minute or per audio token, per video second, depending on the model |
| **Fine-tuning** | Training (per token or per hour) and **hosting** of fine-tuned deployments (hourly) plus inference tokens |
| **Managed compute** | Hourly per accelerator (A100, H100, MI300X) for Foundry managed compute (preview); VM hours for Azure ML online endpoints |
| **Partner models (Marketplace)** | Billed through Azure Marketplace with the provider's terms |
| **Agent Service tools** | Some tools bill separately (for example code interpreter sessions, file search storage, web grounding, Foundry IQ / AI Search) |
| **Other** | AI Search, Storage, Cosmos DB, Application Insights ingestion, API Management, evaluations (model calls by evaluators) |

### Cost attribution per project

| Technique | How |
|---|---|
| **Separate resources** | Cleanest: one Foundry resource per cost centre or app; costs appear per resource in Cost Management |
| **Tags** | Tag resources (cost centre, app, environment); deployments aren't separate Azure resources, so tag at resource level |
| **Separate deployments per team** | Track tokens per deployment via Azure Monitor metrics |
| **Project-level metrics** | Foundry shows token usage per project and agent (Control Plane cost views) |
| **AI gateway token metrics** | `llm-emit-token-metric` with subscription, team or app dimensions for precise chargeback across shared deployments (topic 11 file 07) |
| **Budgets and alerts** | Cost Management budgets per resource group or tag; anomaly alerts |

## How it works on Azure

### Inspect quota and usage

```bash
# Usage vs limit for model quotas in a region (needs Cognitive Services Usages Reader at subscription scope)
az cognitiveservices usage list --location swedencentral -o table

# What models and SKUs can I deploy here?
az cognitiveservices model list --location swedencentral \
  --query "[?model.name=='gpt-5-mini'].{name:model.name, version:model.version, skus:model.skus[].name}" -o json
```

The Foundry portal's **Manage → Quota** pane shows per-deployment usage; **Show all** lists models and regions you haven't deployed yet.

### Estimating cost for a use case

```text
Requests per month            = 200,000
Avg input tokens per request  = 3,000 (system prompt 800 + retrieved chunks 2,000 + question 200)
Avg output tokens per request = 400 (+ reasoning tokens if a reasoning model)
Cached input share            = 25% (repeated system prompt prefix)

Monthly input tokens  = 600M  (150M cached, 450M uncached)
Monthly output tokens =  80M
Cost ≈ 450M × input rate + 150M × cached rate + 80M × output rate
     + AI Search tier + App Insights + gateway + hosting
```

Use the Azure pricing calculator for current rates. Measure real token counts from a prototype (`usage` in responses) instead of guessing.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Region choice | Pick regions with the needed models and deployment types, close to users and data, meeting residency rules. Have a second region planned |
| Deployment type | Global Standard for best availability and price unless residency requires Data Zone or Regional (topic 15 file 04) |
| Pay-as-you-go vs PTU | PAYG for variable or early workloads; PTU for steady, high-volume, latency-sensitive production (topic 15 file 05) |
| Quota planning | Request quota well before launch; spread load across deployments, regions or models; use the gateway to share fairly |
| Chargeback | Separate resources for strict chargeback; gateway token metrics for shared platforms |

## Hands-on

1. Run `az cognitiveservices usage list` for two regions and compare available quota for one model.
2. In the portal, use **Quota → Show all** to find a region where a newly released model is available.
3. Check your subscription's quota tier with the quota tiers API (see the quota article).
4. Build a cost model in the pricing calculator for the example above with two candidate models; compare Global Standard and Data Zone Standard.
5. Create a Cost Management budget on the Foundry resource group with an 80% alert.

## Security, cost and reliability notes

- **Cost:** reasoning tokens, long retrieved contexts and agent tool loops are the usual causes of overspend. Cap output, cache prefixes, trim retrieval and route simple tasks to smaller models.
- **Cost:** soft-deleted resources and idle fine-tuned deployments can keep costing or holding quota; purge and delete them.
- **Reliability:** quota exhaustion looks like an outage (429s). Monitor utilisation per deployment and alert well before limits.
- **Governance:** restrict who can create high-capacity deployments, especially PTU, using RBAC and Azure Policy.

## Common mistakes

- Designing around a model before confirming it's available in the required region and deployment type.
- Launching on default quota without load testing.
- Ignoring cached-token discounts and batch pricing for suitable workloads.
- Assuming project-level separation gives separate bills.
- Forgetting non-model costs (Search, Cosmos DB, App Insights, APIM, evaluations).

## Check yourself

1. What two units define model quota?
2. How is quota pooled for Global Standard deployments after the 2026 change?
3. What are quota tiers, and how does a subscription move between them?
4. What HTTP status indicates rate limiting, and what header tells you when to retry?
5. Name five pricing meters you might see for a Foundry solution.
6. How can you attribute shared deployment costs to teams?

<details>
<summary>Answers</summary>

1. Tokens per minute (TPM) and requests per minute (RPM).
2. Deployments of the same model and version share one quota pool across all regions in the subscription (Data Zone Standard shares one pool per data zone).
3. Seven levels (Free/Tier 0 and Tiers 1–6) that set default quotas; subscriptions auto-upgrade as consumption grows, considering relationship (EA/MCA-E) and payment history; customers can opt out.
4. HTTP 429, with the `retry-after` header (plus `x-ratelimit-*` headers).
5. Any five of: input tokens, cached input tokens, output tokens, batch tokens, PTU hours or reservations, image/audio/video units, fine-tuning training and hosting, managed compute accelerator-hours, Marketplace partner billing, agent tool charges, AI Search, Storage, App Insights, API Management.
6. Use AI gateway token metrics with team or app dimensions, separate deployments per team with Azure Monitor metrics, or separate resources with tags and budgets.

</details>

## References

- [Azure OpenAI quotas and limits](https://learn.microsoft.com/azure/foundry/openai/quotas-limits)
- [Foundry Models quotas and limits](https://learn.microsoft.com/azure/foundry/foundry-models/quotas-limits)
- [Manage and increase quotas](https://learn.microsoft.com/azure/foundry/how-to/quota)
- [Region availability for Foundry Models sold by Azure](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability)
- [Feature availability across regions](https://learn.microsoft.com/azure/foundry/reference/region-support)
- [Plan and manage costs for Foundry](https://learn.microsoft.com/azure/foundry/concepts/manage-costs)
- [Azure OpenAI pricing](https://azure.microsoft.com/pricing/details/azure-openai/)
