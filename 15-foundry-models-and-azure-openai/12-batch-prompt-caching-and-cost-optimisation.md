# Batch, Prompt Caching and Cost Optimisation

## Why it matters

Token costs scale with usage, and a successful AI feature can generate a bill nobody planned for. Foundry gives you several levers that don't change application behaviour much: the **Batch API** (50% cheaper for asynchronous work), **prompt caching** (discounted repeated prefixes), **priority and flex processing** (pay for latency only when you need it), plus disciplined **token budgeting**. Architects use these to hit cost targets; consultants use them to make business cases work.

## Key concepts

### Where the money goes

```text
Cost per request ≈ (uncached input tokens × input price)
                 + (cached input tokens × cached price)  [+ cache-write tokens on GPT-5.6+]
                 + (output tokens + reasoning tokens) × output price
                 + tool charges (search, code interpreter, web grounding) + retries
```

Output and reasoning tokens usually cost several times more than input tokens. Long system prompts, large retrieved contexts and conversation history drive input; verbosity and reasoning effort drive output.

### Batch API

| Aspect | Detail (checked 2026-11) |
|---|---|
| Deployment types | **Global Batch** (`GlobalBatch`) and **Data Zone Batch** (`DataZoneBatch`) |
| Price | **50% less** than Global Standard |
| Turnaround | 24-hour **target** (`completion_window="24h"`); no real-time SLA, may take longer |
| Quota | Separate **enqueued-token** quota, so batch doesn't disturb online traffic; enable **dynamic quota** on batch deployments to reduce failures |
| Input | JSONL file, one request per line with `custom_id`, `method`, `url` (`/v1/chat/completions` or `/v1/responses`) and `body` (with `model` = batch deployment name) |
| Files | Upload with `purpose="batch"`; optional expiry (14–30 days); input file limit 200 MB (1 GB with bring-your-own Blob storage) |
| Lifecycle | Statuses include validating, in progress, finalising, completed, failed, expired, cancelling, cancelled; cancel leaves partial results |
| Output | Output and error JSONL files keyed by `custom_id` (order isn't guaranteed) |
| Limitations | No fine-tuned models in batch; content filtering applies |

Good fits: offline classification and extraction, document summarisation backlogs, synthetic data and evaluation runs, embeddings refreshes (where supported), nightly enrichment.

### Prompt caching

| Rule | Detail |
|---|---|
| Eligibility | Prompts of **at least 1,024 tokens** whose **first 1,024 tokens are identical** to a recent request; enabled by default on supported models (GPT-4o and newer) |
| Granularity | Older models (up to GPT-5.5) match in **128-token increments** after the first 1,024; GPT-5.6+ doesn't use that rounding |
| What's cached | Messages (system, developer, user, assistant), images (same `detail`), tool definitions, structured output schema |
| Visibility | `usage.prompt_tokens_details.cached_tokens` (Chat Completions) / `usage.input_tokens_details.cached_tokens` (Responses); GPT-5.6+ also reports `cache_write_tokens` |
| Pricing | Cached reads are discounted on Standard deployments. Before GPT-5.6, cache writes are free; from GPT-5.6, **cache writes can be charged** (12-hour cache-write charges from November 2026) |
| Provisioned | For models before GPT-6, cached input tokens **don't consume PTU capacity**; GPT-6+ counts cache reads and writes at normalised weights |
| Retention | In-memory: typically cleared after 5–10 minutes idle, always within an hour. **Extended** retention (up to 24 hours) for some models (`prompt_cache_retention: "24h"`; default for gpt-5.5). GPT-5.6+: `prompt_cache_options.ttl` (`30m` default; `12h` on supported Responses deployments) |
| Better hits | `prompt_cache_key` to group requests that share prefixes (GPT-5.6+); explicit **cache breakpoints** (`prompt_cache_breakpoint`) on GPT-5.6+ standard deployments |
| Isolation | Caches aren't shared between Azure subscriptions |

Design prompts for caching: **stable content first** (instructions, tool definitions, few-shot examples, reference documents), **variable content last** (user question, retrieved chunks that change).

### Priority and flex processing

| Tier | Where | Effect |
|---|---|---|
| **Priority processing** | Global Standard and Data Zone Standard (model-dependent) | Higher price per token for faster, more consistent latency with defined latency targets, without PTU commitment |
| **Flex processing** | Global Standard (model-dependent) | Lower price for delay-tolerant requests that can wait for spare capacity |

Both are set per request (a processing tier parameter on supported models). Check the current how-to for parameter names and supported models.

### Token budgeting

| Technique | Saves |
|---|---|
| Right-size the model (mini/nano, SLMs, model router) | Large per-token savings |
| Tune `reasoning_effort` (none or low where possible) | Reasoning tokens |
| Cap output (`max_output_tokens` / `max_completion_tokens`) and ask for concise or structured outputs | Output tokens |
| Trim retrieval: fewer, better chunks (reranking, top-k tuning) | Input tokens |
| Summarise or compact conversation history | Input tokens per turn |
| Cache-friendly prompt ordering | Discounted input |
| Semantic caching at the gateway for repeated questions | Whole calls (topic 11 file 07) |
| Batch for non-interactive work | 50% |
| Per-team token limits and budgets at the gateway | Runaway usage |

## How it works on Azure

### Submit a batch job (Python, keyless)

```python
# tasks.jsonl (one line per request):
# {"custom_id": "doc-001", "method": "POST", "url": "/v1/responses",
#  "body": {"model": "gpt-5-mini-batch", "input": "Classify: 'Invoice overdue by 30 days' → billing|tech|other"}}

batch_file = client.files.create(file=open("tasks.jsonl", "rb"), purpose="batch")
job = client.batches.create(input_file_id=batch_file.id,
                            endpoint="/chat/completions",   # required; the service reads each line's url to pick the API
                            completion_window="24h")

job = client.batches.retrieve(job.id)          # poll periodically
if job.status == "completed":
    results = client.files.content(job.output_file_id).text   # JSONL keyed by custom_id
```

`client` is the OpenAI v1 client with Entra ID (file 02). `gpt-5-mini-batch` is a **Global Batch** deployment name.

### Measure caching

```python
r = client.responses.create(model="<deployment>", instructions=LONG_STABLE_INSTRUCTIONS, input=user_question)
u = r.usage
print("input:", u.input_tokens, "cached:", u.input_tokens_details.cached_tokens, "output:", u.output_tokens)
```

Track the **cache hit ratio** (`cached_tokens / input_tokens`) over time; a sudden drop usually means someone changed the start of the prompt (for example adding a timestamp).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Batch vs real-time | If nobody waits for the answer, batch it |
| Batch vs flex | Batch for big files of work with a 24-hour window; flex for individual delay-tolerant requests |
| Priority vs provisioned | Priority for latency-sensitive traffic without commitment or with variable volume; provisioned for steady high volume |
| Prompt design | Stable prefix first; avoid timestamps, request IDs or user names at the top of prompts |
| Extended caching | Useful for long, repeated contexts across longer gaps; note data stays within zone or region boundaries for Data Zone and Regional types |
| Cache writes on GPT-5.6+ | Use explicit breakpoints and stable prefixes so writes are followed by many reads |

## Hands-on

1. Build a 1,000-line JSONL batch (classification) and run it on a Global Batch deployment; compare cost with the same work on Global Standard.
2. Send 20 requests with a 2,000-token stable system prompt and varied questions; plot `cached_tokens`. Then insert a timestamp at the top of the prompt and observe the hit rate collapse.
3. Measure the same task at reasoning effort `low` vs `high`: quality, latency, output tokens, cost.
4. Add `max_output_tokens` and a "respond in at most 3 bullets" instruction; compare output token counts.
5. Build a monthly cost model for one use case before and after applying these optimisations.

## Security, cost and reliability notes

- **Security:** batch input and output files contain your data; set file expiry and delete files when done; use Blob storage with private networking for large or sensitive batches.
- **Security:** extended prompt caching stores computation on GPU-local storage within the zone or region boundary for Data Zone and Regional types. Consider it in residency reviews.
- **Cost:** monitor cache-write charges on GPT-5.6+ models; poorly structured prompts can now cost more, not just fail to save.
- **Reliability:** batch jobs can fail on enqueued-token limits. Enable dynamic quota and use retries with backoff for large job sequences.

## Common mistakes

- Running offline workloads on real-time deployments at full price.
- Putting dynamic content (dates, user IDs) at the start of prompts and killing cache hits.
- Ignoring reasoning tokens in cost estimates.
- No output caps, so verbose models inflate bills.
- Expecting batch results within minutes, or in input order.

## Check yourself

1. What discount and turnaround does the Batch API offer, and what quota does it use?
2. What's the format of a batch input line?
3. What two conditions make a prompt eligible for caching?
4. How do you see cached tokens in a response?
5. What changed about cache pricing with GPT-5.6 models?
6. When would you use priority processing instead of provisioned throughput?
7. Name five token-budgeting techniques.

<details>
<summary>Answers</summary>

1. 50% less than Global Standard, with a 24-hour target turnaround, using a separate enqueued-token quota.
2. A JSON object with `custom_id`, `method` (POST), `url` (`/v1/chat/completions` or `/v1/responses`) and `body` containing the request with `model` set to the batch deployment name.
3. At least 1,024 tokens long, with the first 1,024 tokens identical to a recent request.
4. `cached_tokens` under `prompt_tokens_details` (Chat Completions) or `input_tokens_details` (Responses).
5. Cache writes can be charged in addition to discounted reads (with 12-hour write charges from November 2026), and new controls (cache keys, breakpoints, TTL) were added.
6. For latency-sensitive traffic when volume is variable or you don't want a PTU commitment.
7. Smaller models or router, lower reasoning effort, output caps, trimmed retrieval, compacted history, cache-friendly prompts, semantic caching, batch, gateway token limits.

</details>

## References

- [Global batch processing](https://learn.microsoft.com/azure/foundry/openai/how-to/batch)
- [Prompt caching](https://learn.microsoft.com/azure/foundry/openai/how-to/prompt-caching)
- [Priority processing](https://learn.microsoft.com/azure/foundry/openai/concepts/priority-processing)
- [Flex processing](https://learn.microsoft.com/azure/foundry/openai/how-to/flex-processing)
- [Deployment types](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)
- [Azure OpenAI pricing](https://azure.microsoft.com/pricing/details/azure-openai/)
- [Plan and manage costs for Foundry](https://learn.microsoft.com/azure/foundry/concepts/manage-costs)
