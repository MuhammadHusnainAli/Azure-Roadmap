# AI Cost Modelling

## Why it matters

AI workloads have the least intuitive cost profiles in the cloud. A chatbot's cost depends on how long conversations are, how many documents are retrieved per answer, which model answers, whether reasoning tokens are used, how often agents call tools and loop, and whether capacity is pay-as-you-go or provisioned. A proof of concept that cost €200 a month can become €200,000 at production scale. Architects and consultants must model **token economics**, compare **provisioned throughput (PTU) with pay-as-you-go**, include **retrieval and agent costs**, and apply levers like **caching**, **batch**, **model routing** and right-sized models, then validate estimates with real telemetry. Pricing details per model live in [Topic 15](../15-foundry-models-and-azure-openai/README.md); this file is about building the model.

## Key concepts

### Cost components of an AI solution

| Component | Driver |
|---|---|
| **Model inference** | Input tokens, cached input tokens, output tokens (including reasoning tokens) × per-model rates; or PTU hours |
| **Embeddings** | Tokens embedded at ingestion and per query |
| **Retrieval** | Azure AI Search tier and units (replicas × partitions), semantic ranker queries, vector storage; or database vector search |
| **Agent runtime** | Additional model calls per step, tool calls, code interpreter sessions, file search storage, hosted agent compute |
| **Data and storage** | Documents, conversation state (Cosmos DB), memory stores |
| **Gateway and hosting** | API Management tier, App Service/Container Apps, Functions |
| **Observability and evaluation** | Application Insights ingestion, evaluation judge tokens, red teaming runs |
| **Safety** | Guardrails are included for Foundry models; separate Content Safety API calls are billed |
| **People and operations** | Platform, evaluation and support effort |

### Token economics

```text
Cost per request ≈ (input_tokens − cached_tokens) × input_rate
                 + cached_tokens × cached_input_rate
                 + output_tokens × output_rate          (reasoning tokens count as output)
Cost per conversation = Σ requests (including agent steps and retries)
Monthly cost = conversations/month × cost per conversation
```

Input grows with system prompt, conversation history, retrieved chunks and tool definitions; output with answer length and reasoning effort. Agents multiply calls: a five-step agent run may cost 5–10× a single chat turn.

### PTU vs pay-as-you-go

| | Pay-as-you-go (Standard, Global, Data Zone) | Provisioned (PTU) |
|---|---|---|
| Billing | Per token | Per PTU per hour; monthly/yearly reservations for discounts |
| Latency | Variable, shared capacity | Predictable |
| Throttling | TPM/RPM quotas | Capacity-bound; 429 or spillover when full |
| Best for | Variable, spiky or low volume | Sustained high volume, latency-sensitive production |

**Break-even**: estimate the tokens per minute your PTU allocation can serve for your prompt/output shape (use the capacity calculator), compute the equivalent pay-as-you-go cost at your expected utilisation, and compare with PTU hourly or reserved cost. Low average utilisation favours pay-as-you-go; high, steady utilisation favours reserved PTU. A common pattern is PTU for the baseline plus **spillover** to standard deployments for peaks.

### Cost levers

| Lever | Effect |
|---|---|
| **Right model** | Smaller/mini/nano models or SLMs for simple tasks; **model router** to pick per request |
| **Prompt caching** | Discounted cached input for repeated prefixes (put stable content first) |
| **Batch API** | ~50% lower price for asynchronous jobs |
| **Shorter context** | Trim history, summarise, retrieve fewer/better chunks |
| **Output limits** | `max_output_tokens`, concise style, lower reasoning effort where acceptable |
| **Agent limits** | Max steps/tool calls per run; avoid loops |
| **Response caching** | Semantic cache in the gateway for repeated questions |
| **Search sizing** | Right tier, scale replicas with load, turn off unused semantic ranker |
| **Distillation/fine-tuning** | Move volume to a cheaper fine-tuned small model (Topic 16) |

## How it works on Azure

Worked example (illustrative rates; use current prices from the pricing page or Retail Prices API):

| Assumption | Value |
|---|---|
| Conversations per month | 200,000 |
| Turns per conversation | 4 |
| Input tokens per turn (system 1,500 + history 1,000 + retrieved 2,000 + question 100) | 4,600 (2,000 cached) |
| Output tokens per turn | 400 |
| Model rates (per 1M tokens) | input €0.25, cached input €0.025, output €2.00 |

```text
Per turn: (2,600 × 0.25 + 2,000 × 0.025 + 400 × 2.00) / 1,000,000 = €0.0015
Per conversation (4 turns): €0.0060
Monthly model cost: 200,000 × €0.0060 ≈ €1,200
Embeddings for queries and ingestion: small; AI Search S1 with 2 replicas: fixed monthly; APIM, hosting, App Insights: fixed + usage
```

Then add agent steps (for example ×3 calls per turn), evaluation (judge tokens on 5% sampled traffic), and growth scenarios (×5, ×20).

Validate with telemetry: platform metrics `ProcessedPromptTokens`, `GeneratedTokens` and cache match rate; APIM token metrics per consumer; Cost Management grouped by meter. Compare cost per conversation actuals with the model monthly (Topic 26 file 09).

Unit economics to report: cost per conversation, cost per resolved case, cost per 1,000 documents processed, against value per outcome ([Topic 22 file 10](../22-evaluation-observability-and-genaiops/10-measuring-business-value.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Model choice** | Evaluate quality per cost; route easy requests to cheaper models. |
| **PTU timing** | Start PAYG; buy PTU reservations once traffic is steady and measured. |
| **Residency vs price** | Global deployments are often cheaper and more available; Data Zone/regional when required. |
| **Context design** | Cache-friendly prompts (stable prefix), fewer but better retrieved chunks. |
| **Budgets and limits** | APIM token limits per consumer, budgets per AI workload, anomaly alerts. |
| **Scenario planning** | Present low/expected/high usage scenarios; AI demand is uncertain. |

## Hands-on

1. Build a token cost model spreadsheet with the formulas above for your use case; include agent multipliers.
2. Measure actual tokens per turn from telemetry for 100 sample conversations; recalibrate.
3. Calculate PTU vs pay-as-you-go break-even at 30%, 60% and 90% utilisation.
4. Restructure a prompt for caching and measure the cache match rate change.
5. Move a nightly summarisation job to the Batch API and compare cost.

## Security, cost and reliability notes

- **Runaway costs** from loops, abuse or bugs: rate limits, token quotas per consumer, max steps, anomaly alerts.
- **Cheaper models can reduce quality or safety**: re-evaluate after switching.
- **Capacity**: PTU guarantees capacity; PAYG may throttle at peak.
- **Price changes**: model prices and new models change often; refresh models quarterly.

## Common mistakes

| Mistake | Fix |
|---|---|
| Modelling one turn, ignoring history and retrieval | Model full conversation context |
| Ignoring reasoning tokens | Include them as output |
| Buying PTU for a pilot | PAYG until usage is steady |
| No per-consumer visibility | Gateway token metrics |
| Forgetting search, evaluation and observability costs | Full solution model |

## Check yourself

1. List the cost components of an AI solution.
2. Write the per-request token cost formula.
3. When is PTU cheaper than pay-as-you-go?
4. Name six cost levers for AI workloads.
5. Why do agents cost more than chat?
6. How do you validate an AI cost model after launch?

<details>
<summary>Answers</summary>

1. Model inference, embeddings, retrieval, agent runtime, data/storage, gateway/hosting, observability/evaluation, safety APIs, people/operations.
2. (Uncached input × input rate) + (cached input × cached rate) + (output including reasoning × output rate).
3. At sustained high utilisation where reserved PTU cost is below the equivalent token cost.
4. Any six: right model/router, prompt caching, Batch API, shorter context, output limits, agent step limits, response caching, search sizing, distillation/fine-tuning.
5. Each run involves multiple model calls and tool invocations with growing context.
6. Compare telemetry (token metrics, cache rates, APIM per-consumer usage) and Cost Management data with the model; track cost per outcome.

</details>

## References

- [Provisioned throughput concepts](https://learn.microsoft.com/azure/foundry/openai/concepts/provisioned-throughput)
- [Provisioned throughput onboarding and costs](https://learn.microsoft.com/azure/foundry/openai/how-to/provisioned-throughput-onboarding)
- [Prompt caching](https://learn.microsoft.com/azure/foundry/openai/how-to/prompt-caching)
- [Batch API](https://learn.microsoft.com/azure/foundry/openai/how-to/batch)
- [Azure AI Search tiers](https://learn.microsoft.com/azure/search/search-sku-tier)
- [Well-Architected AI design principles](https://learn.microsoft.com/azure/well-architected/ai/design-principles)
