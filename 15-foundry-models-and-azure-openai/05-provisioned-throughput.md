# Provisioned Throughput

## Why it matters

Pay-per-token deployments share capacity with other customers. At scale that means variable latency and the risk of 429s at the worst moment. **Provisioned throughput** reserves model-processing capacity for your deployment, measured in **provisioned throughput units (PTUs)**, giving predictable throughput and latency. It's also a major financial commitment. Sizing, reservations, spillover and the break-even with pay-as-you-go come up in nearly every enterprise production design and procurement discussion.

## Key concepts

### What a PTU is

| Property | Detail (checked 2026-11) |
|---|---|
| **Definition** | A fixed amount of model-processing capacity, held for your deployment whether or not you send requests |
| **Model-independent quota** | PTU quota isn't bought per model; the same quota can deploy any supported model |
| **Scope** | PTU quota is per **subscription, region and deployment type** (Global Provisioned, Data Zone Provisioned and Regional Provisioned are separate pools) |
| **Throughput per PTU varies by model** | Heavier models deliver fewer tokens per minute per PTU; each model publishes an "input TPM per PTU" value |
| **Minimums and increments** | Each model has a minimum PTU deployment size and increment, per deployment type, published in the sizing article |
| **Quota ≠ capacity** | Quota is a policy limit; **capacity** is actual available hardware at deployment time. Having quota doesn't guarantee capacity, so check before you commit |

### Deployment types

| Type | SKU | Routing |
|---|---|---|
| Global Provisioned | `GlobalProvisionedManaged` | Across Azure regions (highest availability of capacity) |
| Data Zone Provisioned | `DataZoneProvisionedManaged` | Within the data zone (US or EU) |
| Regional Provisioned | `ProvisionedManaged` | Within the deployment's region or geography |

### Sizing

Three inputs drive the PTU count:

| Input | Why |
|---|---|
| **Request shape**: peak requests per minute, average input tokens, average output tokens | Defines load |
| **Output-to-input ratio** | Output tokens cost more capacity than input; for GPT-4.1 and later, the ratio generally matches the model's Global Standard price ratio (with documented exceptions) |
| **Cache rate** | For models before GPT-6, **cached input tokens don't consume PTU capacity**. For GPT-6 and later, cache reads and writes count at normalised weights |

The method converts expected tokens into a single **normalised TPM**, then divides by the model's input TPM per PTU, rounded up to the model's minimum and increment. Use the **capacity calculator** in the Foundry portal and validate with a load test.

```text
Example (illustrative only, use the model's published values):
  Peak 300 RPM × (2,000 input + 400 output tokens)
  Output-to-input weight = 4  → normalised tokens per request = 2,000 + 400×4 = 3,600
  Normalised TPM = 300 × 3,600 = 1,080,000
  Model input TPM per PTU = 2,500 (hypothetical) → 432 PTU → round up to the model's increment
```

Always size on **peak**, not average, and add headroom. Over-utilisation returns 429s.

### Behaviour at full utilisation

- Provisioned deployments expose a **utilisation** metric (Provisioned-Managed Utilization). When it reaches 100%, new requests receive **429** with a `retry-after` header, rather than queueing indefinitely.
- That makes behaviour predictable: you know when you're out of capacity, and you can **spill over** or retry.

### Spillover

**Spillover** automatically routes overflow from a provisioned deployment to a **standard deployment in the same Foundry resource** when the provisioned deployment returns non-200 responses (for example 429 when PTUs are exhausted).

| Point | Detail |
|---|---|
| Scope | All Azure OpenAI models that support provisioned throughput support spillover; models from other providers (for example DeepSeek, Llama) currently don't |
| Configuration | For all requests on a deployment, or per request with the `x-ms-spillover-deployment` header |
| Billing | Spilled requests are billed at the standard deployment's pay-as-you-go rate |
| Alternative | API Management priority backend pools (topic 11 file 07) give similar behaviour across resources and regions |

### Billing: hourly vs reservations

| Mode | How | When |
|---|---|---|
| **Hourly** | $/PTU/hour for every PTU deployed, from creation to deletion, regardless of tokens used | Short-term needs: benchmarking, events, validation |
| **Azure Reservations** | 1-month or 1-year commitment on the PTU meter for a discounted effective rate; purchased per deployment type and scoped to subscriptions or resource groups | Sustained production |

Important points from Microsoft's guidance:

- Reservations are a **billing discount, not a capacity guarantee**. Create the deployment first (confirming capacity), then buy the reservation.
- Reservations and deployments are **loosely coupled**: the discount applies to matching PTU usage in scope.
- Don't plan to scale provisioned deployments up and down with traffic. Capacity might not be available when you scale back up, and continuous hourly billing at high utilisation usually exceeds reservation pricing.

### When provisioned beats pay-as-you-go

| Factor | Favours provisioned | Favours pay-as-you-go |
|---|---|---|
| Traffic | Predictable, sustained, high | Spiky, low or unknown |
| Latency | Strict and consistent latency needed | Best effort acceptable |
| Utilisation | You can keep PTUs busy (high average utilisation) | Utilisation would be low most of the day |
| Commitment | Stable workload for months | Experimental or changing |

Break-even: compare `PTUs × hours × reserved rate` against the token bill for the same workload at pay-as-you-go rates. If average utilisation is high, provisioned with a reservation is usually cheaper. If it sits idle overnight, a mix of a smaller PTU base plus spillover to pay-as-you-go often wins. **Priority processing** is an alternative when you need better latency without commitment.

## How it works on Azure

### Check capacity and deploy

```bash
# Create a provisioned deployment (PTU count must respect the model's minimum and increment)
az cognitiveservices account deployment create -g rg-foundry -n fdy-contoso-prod \
  --deployment-name gpt-5-ptu --model-format OpenAI --model-name gpt-5 --model-version 2025-08-07 \
  --sku-name GlobalProvisionedManaged --sku-capacity <ptu-count>
```

Use the Foundry portal deployment wizard or the **model capacities API** to see where capacity exists before deploying. If your region lacks capacity, the portal suggests alternatives.

### Spillover per request (REST header)

```bash
curl -s https://<res>.openai.azure.com/openai/v1/chat/completions \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "x-ms-spillover-deployment: gpt-5-standard" \
  -d '{"model":"gpt-5-ptu","messages":[{"role":"user","content":"hello"}]}'
```

### Monitor

| Metric | Use |
|---|---|
| Provisioned-Managed Utilization (V2) | Keep peak below 100%; alert at 80–90% |
| Time to response / latency | Validate latency targets |
| Requests by status code | Watch 429s and spillover volume |
| Reservation utilisation (Cost Management) | Make sure you're using what you pay for |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Global vs Data Zone vs Regional provisioned | Global for best capacity availability; Data Zone or Regional only when residency requires it (capacity is scarcer) |
| PTU base + overflow | Size PTUs for typical peak; spill over to pay-as-you-go for rare bursts |
| One big vs several deployments | Separate deployments per critical app give isolation; one shared deployment gives better utilisation. Use the AI gateway to share fairly |
| Reservation term | 1-month for early production confidence; 1-year once usage is stable |
| Model upgrades | Provisioned deployments **aren't auto-upgraded**; plan in-place or side-by-side migrations before retirement (file 14) |

## Hands-on

1. Use the capacity calculator for a workload with 200 RPM, 3,000 input and 500 output tokens, and a 30% cache rate, on two different models; compare PTU counts.
2. Create a small hourly provisioned deployment (minimum size) in a test subscription and load-test it with a script until utilisation hits 100%; observe 429s.
3. Configure spillover to a Global Standard deployment and repeat; compare error rates and cost.
4. Build a break-even spreadsheet: pay-as-you-go tokens vs PTU hourly vs PTU reserved at 30%, 60% and 90% average utilisation.
5. Delete the provisioned deployment afterwards; hourly billing continues while it exists.

## Security, cost and reliability notes

- **Cost:** PTUs bill per hour whether used or not. Forgotten test deployments are expensive; restrict who can create provisioned deployments.
- **Cost:** buy reservations only after the deployment exists and utilisation data supports it.
- **Reliability:** provisioned capacity is regional. For high availability, combine deployments in two regions (or Global Provisioned) with a gateway and spillover.
- **Reliability:** capacity released by deleting or downsizing may not be available again later.

## Common mistakes

- Buying a reservation before confirming capacity can be deployed.
- Sizing on average load, so the deployment saturates at peak.
- Treating quota as guaranteed capacity.
- Expecting provisioned deployments to auto-upgrade at model retirement.
- Scaling PTUs up and down daily to "save money" and losing capacity.

## Check yourself

1. What is a PTU, and is PTU quota tied to a specific model?
2. What's the difference between PTU quota and capacity?
3. What three factors drive PTU sizing?
4. What does spillover do, and how do you trigger it per request?
5. Do Azure Reservations guarantee capacity?
6. When does provisioned throughput beat pay-as-you-go?

<details>
<summary>Answers</summary>

1. A unit of reserved model-processing capacity. Quota is model-independent but scoped per subscription, region and deployment type.
2. Quota is a policy limit on how many PTUs you may deploy; capacity is the actual hardware available when you create the deployment. Quota doesn't guarantee capacity.
3. Request shape (RPM, input and output tokens), the model's output-to-input ratio, and the prompt cache rate.
4. It routes overflow (for example 429s) from a provisioned deployment to a standard deployment in the same resource; per request with the `x-ms-spillover-deployment` header.
5. No. They're a billing discount; create the deployment first to secure capacity.
6. For predictable, sustained, high-volume workloads with strict latency needs and high average utilisation, especially with a reservation.

</details>

## References

- [Provisioned throughput concepts](https://learn.microsoft.com/azure/foundry/openai/concepts/provisioned-throughput)
- [Determine PTU sizing for a workload](https://learn.microsoft.com/azure/foundry/openai/how-to/provisioned-throughput-sizing)
- [Get started with provisioned deployments](https://learn.microsoft.com/azure/foundry/openai/how-to/provisioned-get-started)
- [Manage traffic with spillover](https://learn.microsoft.com/azure/foundry/openai/how-to/spillover-traffic-management)
- [Provisioned throughput billing and reservations](https://learn.microsoft.com/azure/foundry/openai/concepts/provisioned-throughput-billing)
- [Monitor Azure OpenAI](https://learn.microsoft.com/azure/foundry-classic/openai/how-to/monitor-openai)
