# Capacity and Performance Planning

## Why it matters

AI performance problems look different from classic ones: the app is fine at 50 users, then latency triples and 429s appear at 300 because the model deployment hit its tokens-per-minute quota; a PTU deployment runs at 100% utilisation at 9 a.m. and idles overnight; an agent that calls four tools and two models takes 25 seconds; a RAG answer is slow because the prompt carries 20,000 tokens of context. Capacity and performance planning for AI means modelling **throughput in tokens**, setting **latency budgets** across every hop, choosing **PTU vs pay-as-you-go** capacity, and **load testing** with realistic prompts before go-live.

## Key concepts

### Throughput model

| Input | Example |
|---|---|
| Peak concurrent users | 1,000 |
| Requests per user per minute at peak | 0.5 |
| Model calls per request (agents/tools) | 3 |
| Input tokens per call (system + history + context + query) | 3,500 |
| Output tokens per call | 400 |

```text
Calls/min = 1,000 × 0.5 × 3 = 1,500
Tokens/min (TPM) ≈ 1,500 × (3,500 + 400) ≈ 5.85M TPM at peak
```

Compare with deployment quotas (TPM/RPM for standard deployments) and PTU capacity (use the Foundry capacity calculator with your input/output shape; output tokens weigh more than input in PTU sizing).

### Latency budget

| Hop | Typical budget (example) |
|---|---|
| Client → app (network, auth) | 50–100 ms |
| Retrieval (query rewrite + hybrid search + semantic ranking) | 200–600 ms |
| Model time to first token | 300–1,500 ms (depends on model, reasoning effort, input size) |
| Generation (streamed) | output tokens ÷ tokens per second |
| Tools (each) | 100–2,000 ms |
| Agent orchestration overhead | Per step |
| Guardrails | ~50–100 ms per intervention point |

Key metrics: **time to first token (TTFT)**, time between tokens, total response time; p50/p95. Streaming improves perceived latency.

### Capacity options

| Option | Behaviour | Fit |
|---|---|---|
| **Standard (pay-as-you-go)** Global / Data Zone / regional | Shared capacity, quota-limited, variable latency | Variable or modest load |
| **Provisioned throughput (PTU)** Global / Data Zone / regional | Reserved capacity, predictable latency; 429 or spillover when full | High, steady, latency-sensitive load |
| **Spillover** | PTU overflow to standard | Peaks above PTU |
| **Batch** | Asynchronous, cheaper, 24-hour target | Non-interactive workloads |
| **Priority processing / service tiers** | Pay more for lower latency where offered | Latency-critical paths (check availability) |

### Performance levers

| Lever | Effect |
|---|---|
| Smaller/faster models for simple steps; routing | Lower latency and cost |
| Lower reasoning effort where acceptable | Faster TTFT |
| Fewer/better retrieved chunks; prompt caching | Fewer input tokens, faster |
| Shorter outputs (`max_output_tokens`, concise instructions) | Faster completion |
| Parallel tool calls / concurrent agents | Lower end-to-end latency |
| Streaming | Better perceived latency |
| Semantic caching in the gateway | Instant responses for repeats |
| Colocation of app, search and model regions | Lower network latency |

### Load testing AI

Use realistic prompts and conversation lengths (not "hello"), mix of intents, think time, and streaming clients. Measure TTFT, total latency, 429 rate, tokens/min, PTU utilisation and cost per request. Azure Load Testing (JMeter/Locust) can drive AI endpoints through the gateway (Topic 27 file 08).

## How it works on Azure

Capacity plan worksheet:

```text
1. Measure tokens per call from a pilot (telemetry: ProcessedPromptTokens, GeneratedTokens).
2. Forecast peak calls/min from user and agent-step estimates.
3. Compute peak TPM; compare to quota; request increases or plan PTUs (capacity calculator).
4. Decide: PTU for baseline (e.g., 70% of peak), spillover to Data Zone standard for the rest.
5. Load test at 1×, 1.5× and 2× peak through the gateway; record TTFT p95, 429 %, PTU utilisation.
6. Set alerts: PTU utilisation > 90%, 429 > 2%, TTFT p95 > budget (Topic 26 file 09).
7. Review monthly; adjust PTUs and routing.
```

Locust task sketch for a chat endpoint:

```python
from locust import HttpUser, task, between
import random
PROMPTS = open("realistic_prompts.txt").read().splitlines()

class ChatUser(HttpUser):
    wait_time = between(5, 20)
    @task
    def ask(self):
        self.client.post("/api/chat", json={"message": random.choice(PROMPTS), "stream": False},
                         headers={"Authorization": f"Bearer {self.environment.parsed_options.token}"})
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **PTU sizing** | Size for sustained baseline, not absolute peak; spill to standard. |
| **Model choice per step** | Fast/cheap models for routing, classification and extraction; larger models only where quality needs them. |
| **Latency vs quality** | Reasoning models improve quality but increase latency; set per-journey budgets. |
| **Quota planning** | Request quotas early, including DR regions. |
| **Context size** | Larger context improves recall but increases latency and cost; evaluate. |
| **Async UX** | For long agent tasks, use async patterns (notifications) instead of blocking chat. |

## Hands-on

1. Measure tokens per call for 100 pilot conversations.
2. Build the throughput model and compute peak TPM.
3. Use the capacity calculator to size PTUs; design spillover.
4. Load test with realistic prompts at 2× peak; record metrics.
5. Optimise one hop (fewer chunks or smaller model) and re-measure latency and quality.

## Security, cost and reliability notes

- **Load tests** against production models consume quota and money; use dedicated test deployments.
- **Throttling** is a reliability risk; alerts and fallbacks are mandatory.
- **Cost**: PTU reservations commit spend; validate with real utilisation (Topic 29 file 06).

## Common mistakes

| Mistake | Fix |
|---|---|
| Sizing by requests instead of tokens | Token-based model |
| Load testing with trivial prompts | Realistic prompts and conversation lengths |
| PTU sized for absolute peak | Baseline + spillover |
| One large model for every step | Route by task |
| No latency budget | Per-hop budgets and p95 targets |

## Check yourself

1. How do you estimate peak tokens per minute?
2. List the hops in an AI latency budget.
3. Compare standard, PTU, spillover and batch capacity.
4. Name six performance levers.
5. What makes AI load testing different?
6. Why size PTUs for baseline rather than peak?

<details>
<summary>Answers</summary>

1. Peak users × requests per user per minute × model calls per request × (input + output tokens per call).
2. Client/app network and auth, retrieval, model TTFT, generation, tools, orchestration overhead, guardrails.
3. Standard: shared, quota-limited, variable latency; PTU: reserved, predictable; spillover: PTU overflow to standard; batch: async and cheaper.
4. Any six: smaller models/routing, lower reasoning effort, fewer chunks/prompt caching, shorter outputs, parallel tool calls, streaming, semantic caching, colocation.
5. Realistic prompts, conversation lengths and streaming; measuring TTFT, 429s, tokens/min and PTU utilisation.
6. To avoid paying for idle reserved capacity while using spillover for peaks.

</details>

## References

- [Provisioned throughput concepts](https://learn.microsoft.com/azure/foundry/openai/concepts/provisioned-throughput)
- [Get started with provisioned deployments](https://learn.microsoft.com/azure/foundry/openai/how-to/provisioned-get-started)
- [Performance and latency](https://learn.microsoft.com/azure/foundry/openai/how-to/latency)
- [AI workload operations](https://learn.microsoft.com/azure/well-architected/ai/operations)
- [Azure Load Testing](https://learn.microsoft.com/azure/app-testing/load-testing/overview-what-is-azure-load-testing)
