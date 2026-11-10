# Model Router and Model Selection

## Why it matters

With dozens of capable models available, "which model?" is a recurring engineering and business decision. The answer changes every few months, differs per task, and trades quality against latency and cost. Foundry helps in two ways. **Benchmarks, leaderboards and evaluations** support deliberate selection, and the **model router** makes the choice per request automatically. Used well, they cut cost substantially without hurting quality. Used blindly, they cause unpredictable behaviour.

## Key concepts

### Selection criteria

| Criterion | What to measure |
|---|---|
| **Quality** | Task-specific accuracy, groundedness, instruction following, tool-call accuracy, safety, on **your** evaluation set |
| **Latency** | Time to first token, total time, p95/p99 under load |
| **Cost** | Cost per completed task: input, output and reasoning tokens × prices, plus retries |
| **Capabilities** | Context length, modalities, tools, structured outputs, reasoning controls, Responses API support |
| **Availability** | Regions, deployment types, quota, provisioned capacity |
| **Compliance** | Sold by Azure vs partner, data processing location, terms |
| **Lifecycle** | Retirement date, upgrade path (file 14) |

### Benchmarks and leaderboards in Foundry

The model catalogue includes **model leaderboards** and **benchmark** views:

| View | Shows |
|---|---|
| Quality leaderboards | Aggregate quality indices and scores on public benchmarks (reasoning, knowledge, maths, coding) |
| Safety | Results on safety benchmarks (harmful content, jailbreak susceptibility) |
| Cost | Estimated price per token or per index point |
| Throughput and latency | Measured tokens per second and time to first token |
| Trade-off charts | Quality vs cost, quality vs throughput |
| Compare | Side-by-side model comparison and scenario leaderboards |

Public benchmarks are a starting filter, not the decision. They can be contaminated (models trained on test data), may not reflect your domain, and don't measure your prompts, tools or data. Use them to shortlist, then run **Foundry evaluations** on your own dataset (topic 22).

### Model router

**Model router** is a Microsoft-trained routing model that you deploy like any model (`model-router`). For each request it analyses the prompt (complexity, reasoning need, task type) and forwards it to the most suitable underlying model.

| Aspect | Detail (checked 2026-11) |
|---|---|
| Current version | `2025-11-18` (active, updated in place with new underlying models); older versions `2025-08-07` and `2025-05-19` are frozen |
| Deployment types | Global Standard and Data Zone Standard (honours data zone boundaries) |
| Routing pool | Models from OpenAI (for example GPT-6 Astra, GPT-5.x, o4-mini, GPT-4.1, gpt-oss), Anthropic Claude, xAI Grok, DeepSeek, Meta Llama and Fireworks-hosted models |
| Deploying underlying models | Not needed, **except Claude**, which you deploy first from the catalogue |
| **Routing modes** | **Balanced** (default: picks the cheapest model within about 1–2% of the best quality for that prompt), **Cost** (wider band of about 5–6%), **Quality** (highest-quality model, ignoring cost) |
| **Model subset** | Choose which models may be used (for cost, compliance, capabilities); new models aren't added automatically |
| Automatic failover | Retries on the next most appropriate model; your subset doubles as the fallback set, so include at least two models |
| Tools and agents | Supports agentic scenarios with tools in Foundry Agent Service across eligible models |
| Prompt caching | Works when consecutive requests land on the same underlying model; optional **session affinity** (preview) keeps related Chat Completions turns on one model |
| Inputs | Text and images (routing decided on text only); no audio |
| Context window | **Limited by the smallest underlying model** in the pool. Use a subset of larger-context models if needed |
| Billing | Router charge on input, plus the underlying model's usage |
| Governance | Respects the built-in Azure Policy for allowed model deployments within the subset |
| Observability | The response's `model` field shows which underlying model answered; monitoring shows ordered attempts and fallbacks |

### When to use the router vs choosing explicitly

| Use the model router | Choose a model explicitly |
|---|---|
| Mixed traffic: simple and complex requests from the same app | Narrow, uniform task where one model is clearly best |
| Cost pressure on broad chat or assistant workloads | Strict reproducibility (same model every time) and stable behaviour |
| Teams without bandwidth to keep re-benchmarking new models | Regulatory need to document the exact model used per decision |
| Need for automatic failover across providers | Very long contexts beyond the smallest router model (unless subsetting) |
| | Provisioned throughput (router isn't a provisioned SKU) |

Alternatives for routing you control: an **SLM-first cascade** (topic 13 file 06), an **AI gateway** that routes by product or header (topic 11 file 07), or application logic that picks a model per task type.

## How it works on Azure

### Deploy and call the model router

```bash
az cognitiveservices account deployment create -g rg-foundry -n fdy-contoso-dev \
  --deployment-name model-router --model-format OpenAI --model-name model-router --model-version 2025-11-18 \
  --sku-name GlobalStandard --sku-capacity 100
```

Routing mode and model subset are set in a **custom deployment** (portal or deployment properties).

```python
r = client.chat.completions.create(            # client: OpenAI v1 with Entra ID (file 02)
    model="model-router",                      # the router deployment
    messages=[{"role": "user", "content": "Prove that the sum of two even numbers is even."}])
print("answered by:", r.model)                 # underlying model chosen for this request
print(r.choices[0].message.content)
```

### A selection workflow

```text
1. Define the task, quality bar and constraints (latency, cost per task, residency)
2. Shortlist 3–5 models from leaderboards, filtered by region/deployment type availability
3. Build an evaluation set (50–200 real examples with expected outputs)
4. Run Foundry evaluations for each candidate (+ model router in Balanced and Cost modes)
5. Compare quality, latency (p95), and cost per task; check safety evaluations
6. Choose: a single model, the router with a subset, or a cascade
7. Re-run the evaluation when new models launch or before upgrades (automate in CI)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Router mode | Start with Balanced; move to Cost if evaluations show no quality drop; use Quality for critical tasks |
| Subset | Restrict to approved models (compliance), to models that support your tools and structured outputs, and to large-context models when needed |
| Auto-update of the router | Auto-updating changes the underlying pool and can change cost and behaviour; pin and re-evaluate if predictability matters |
| Reproducibility | Log the answering model per request for audit; route regulated decisions to a fixed model |
| Benchmarks vs evaluations | Benchmarks to shortlist, your evaluations to decide |

## Hands-on

1. Open the model leaderboard and shortlist three models for a summarisation task by quality vs cost.
2. Create a 50-example evaluation set and run Foundry evaluations on the three models.
3. Deploy the model router (Balanced) and run the same evaluation; record which models it used and the total cost.
4. Repeat with Cost mode and a subset of two models; compare quality and cost.
5. Send a 300,000-token prompt to the router without a subset and observe the context error; fix it with a large-context subset.

## Security, cost and reliability notes

- **Security and compliance:** the router may send prompts to models from different providers. Use a model subset to keep traffic within approved models (and Data Zone Standard for residency).
- **Cost:** router savings depend on the traffic mix; measure cost per task before and after.
- **Reliability:** automatic failover improves resilience, but only within the subset; include at least two models.
- **Quality:** model mix changes can shift tone and format. Use structured outputs and evaluation gates.

## Common mistakes

- Choosing models from public leaderboards alone.
- Using the router for a workload that needs the same model every time.
- Forgetting to deploy Claude models before including them in the router.
- Ignoring the router's smallest-model context limit.
- Not logging which underlying model produced each answer.

## Check yourself

1. What are the three routing modes and how do they differ?
2. Which underlying models must you deploy separately for the router?
3. What limits the router's effective context window, and how do you work around it?
4. How do you find out which model answered a router request?
5. Why shouldn't public benchmarks decide your model choice?
6. Name two alternatives to the model router for routing.

<details>
<summary>Answers</summary>

1. Balanced (default) picks the most cost-effective model within a small quality band (about 1–2%) of the best; Cost uses a wider band (about 5–6%) for more savings; Quality always picks the highest-quality model regardless of cost.
2. Anthropic Claude models.
3. The smallest underlying model's context window; use a model subset containing only models with larger context windows (or shorten prompts).
4. Read the `model` field in the response (and use router monitoring for attempt details).
5. They may not reflect your domain, prompts, tools or data, can be contaminated, and ignore your latency and cost constraints.
6. An SLM-first cascade, AI gateway routing (for example by product or header), or application logic that chooses models per task type.

</details>

## References

- [Model router concepts](https://learn.microsoft.com/azure/foundry/openai/concepts/model-router)
- [How to use model router](https://learn.microsoft.com/azure/foundry/openai/how-to/model-router)
- [Evaluate model router for your workload](https://learn.microsoft.com/azure/foundry/openai/how-to/evaluate-model-router)
- [Model leaderboards in Foundry](https://learn.microsoft.com/azure/foundry/concepts/model-benchmarks)
- [Observability and evaluation in Foundry](https://learn.microsoft.com/azure/foundry/concepts/observability)
- [Govern model router deployments with Azure Policy](https://learn.microsoft.com/azure/foundry/how-to/model-router-policy)
