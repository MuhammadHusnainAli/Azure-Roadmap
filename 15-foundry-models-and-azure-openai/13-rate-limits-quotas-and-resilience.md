# Rate Limits, Quotas and Resilience

## Why it matters

The most common production incident in AI apps isn't a bug. It's **HTTP 429 Too Many Requests**: a burst of traffic exceeds the deployment's tokens-per-minute (TPM) or requests-per-minute (RPM) limit, or a region has a capacity problem. Users see errors, agents fail mid-task, and teams scramble for quota. Resilient AI systems combine correct quota planning, client-side retries with backoff, load spreading across deployments and regions, and graceful degradation.

## Key concepts

### How limits work

| Concept | Detail |
|---|---|
| **TPM** | Tokens per minute assigned to a deployment (its "capacity" for standard types, in units of 1,000 TPM) |
| **RPM** | Requests per minute, tied to TPM by a ratio that **varies by model and deployment type** (topic 14 file 10) |
| **Evaluation window** | Limits are enforced over short intervals (not just per full minute), so bursts can be throttled even when the minute's average is fine |
| **Token estimation** | Requests are counted using prompt tokens plus an estimate of output based on `max_tokens`/`max_output_tokens`, so very large caps can throttle sooner |
| **Quota pools** | Subscription-level pools (Global Standard shared across regions for the same model and version; Data Zone per zone) and quota tiers (Free and Tiers 1–6) |
| **Provisioned** | 429 when utilisation reaches 100% (file 05) |
| **Batch** | Separate enqueued-token quota |

### What a 429 tells you

| Signal | Use |
|---|---|
| Status **429** | Rate limited (TPM, RPM or provisioned utilisation) |
| `retry-after` / `retry-after-ms` headers | How long to wait before retrying |
| `x-ratelimit-remaining-tokens`, `x-ratelimit-remaining-requests` | Remaining budget in the current window (for proactive throttling) |
| Status **5xx**, timeouts | Transient service errors: retry with backoff, then fail over |
| Status **400** with `content_filter` | Not retryable as-is |
| Status **410 Gone** | Model version retired: fix the deployment, don't retry |

### Retries with exponential backoff and jitter

- The OpenAI and Azure SDKs retry 429 and 5xx automatically (OpenAI Python default `max_retries=2`), honouring `retry-after` where provided. Increase retries and set timeouts deliberately.
- Use **exponential backoff with jitter** to avoid synchronised retry storms.
- Make retried operations **idempotent**, especially agent tools with side effects.
- Bound total retry time to stay within user-facing latency budgets, then degrade gracefully.

### Resilience patterns

| Pattern | How on Azure |
|---|---|
| **Spread load across deployments** | Several deployments (and models) behind a load balancer: APIM backend pools (round-robin, weighted, priority) |
| **Multi-region back ends** | Foundry resources in two or more regions; AI gateway priority pools with circuit breakers; Front Door for the app tier |
| **PTU first, PAYG overflow** | Provisioned deployment + spillover (same resource) or gateway priority pools (across resources) |
| **Circuit breaker** | APIM backend circuit breaker honouring `Retry-After` |
| **Queue-based load levelling** | Put non-interactive work on Service Bus and consume within quota (topic 11 file 10) |
| **Fallback models** | Model router automatic failover, or explicit fallback to a smaller model when the primary is throttled |
| **Graceful degradation** | Cached answers, "try again shortly" messages, reduced features (no reasoning, shorter answers) |
| **Client-side rate limiting** | Token bucket in your service using the `x-ratelimit-*` headers |

Global Standard and Data Zone Standard route dynamically, but an interruption in the resource's primary region can still affect traffic through that resource, so multi-region designs remain important for critical apps. Foundry doesn't fail over automatically between your resources.

## How it works on Azure

### Configure SDK retries and timeouts

```python
from openai import OpenAI
client = OpenAI(base_url="https://<res>.openai.azure.com/openai/v1/", api_key=tp,
                max_retries=6, timeout=60)   # SDK retries 429/5xx with backoff, honouring retry-after
```

### Explicit backoff with jitter (when you need custom control)

```python
import random, time
from openai import RateLimitError, APIStatusError, APITimeoutError

def call_with_backoff(fn, max_attempts=6, base=1.0, cap=30.0):
    for attempt in range(max_attempts):
        try:
            return fn()
        except RateLimitError as e:
            retry_after = e.response.headers.get("retry-after-ms")
            delay = (int(retry_after) / 1000) if retry_after else min(cap, base * 2 ** attempt)
        except (APITimeoutError, APIStatusError) as e:
            if isinstance(e, APIStatusError) and e.status_code < 500:
                raise                                   # don't retry 4xx other than 429
            delay = min(cap, base * 2 ** attempt)
        time.sleep(delay * random.uniform(0.5, 1.5))    # jitter
    raise RuntimeError("model call failed after retries")
```

### Multi-region topology

```text
Users ─► Front Door ─► App (Container Apps, 2 regions)
                         │
                         ▼
               APIM AI gateway (zone-redundant, or one per region)
               backend pool "chat":
                 priority 1: PTU deployment (Sweden Central)
                 priority 2: Global Standard (Sweden Central)  weight 50
                 priority 2: Global Standard (France Central)  weight 50
               circuit breaker on 429/5xx, acceptRetryAfter = true
```

### Monitor and alert

| Metric or log | Alert when |
|---|---|
| Requests by status code (429 rate) | Above a small percentage for 5 minutes |
| Processed / generated tokens vs deployment TPM | Above 80% of capacity |
| Provisioned utilisation | Above 90% |
| Time to response (p95) | Above target |
| Gateway token metrics per consumer | A consumer exceeds its budget |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Single vs multiple deployments | Multiple deployments (and regions) for critical workloads; single for low-risk internal tools |
| Retry in client vs gateway | Short client retries plus gateway failover; avoid long retry chains at both layers (retry multiplication) |
| Quota allocation | Give each critical app its own deployment or gateway token limit so one noisy app can't starve others |
| Data residency vs resilience | Data Zone or Regional constraints limit failover options; plan multiple regions within the same zone or geography |
| Degradation | Decide in advance what users get when models are unavailable |

## Hands-on

1. Set a deployment to a small TPM (for example 1K) and send a burst; log status codes and `retry-after-ms`.
2. Compare SDK default retries with `max_retries=6` under the same burst.
3. Deploy the same model in two regions and route through APIM with a priority pool and circuit breaker; disable one deployment and confirm failover.
4. Put a batch of 500 classification tasks on a Service Bus queue and process them within quota with a capped consumer.
5. Create Azure Monitor alerts for 429 rate and token utilisation.

## Security, cost and reliability notes

- **Reliability:** test failure paths (throttling, regional outage, retired model) regularly; chaos-test the gateway.
- **Reliability:** use separate deployments for batch-like backend jobs and interactive traffic.
- **Cost:** extra deployments in other regions cost nothing until used (pay-as-you-go), but PTUs do. Use PAYG for standby capacity.
- **Security:** failover regions must meet the same residency, networking and identity requirements as the primary.

## Common mistakes

- No retry handling, or infinite retries without backoff.
- Retrying non-retryable errors (400 content filter, 410 retired).
- Huge `max_tokens` caps that make requests count heavily against TPM.
- One deployment shared by all apps with no per-app limits.
- Assuming Global Standard means "no regional outages ever".

## Check yourself

1. What does an HTTP 429 from a Foundry deployment mean, and which headers help you respond?
2. Why can you get 429s even when average per-minute usage is below the limit?
3. Which errors should you not retry?
4. Name four resilience patterns for model access.
5. How does `max_tokens` affect rate limiting?
6. Why avoid retries at both the client and the gateway with long chains?

<details>
<summary>Answers</summary>

1. Rate limit exceeded (TPM, RPM or provisioned utilisation); `retry-after`/`retry-after-ms` and `x-ratelimit-remaining-*` headers.
2. Limits are evaluated over short intervals, so bursts within a minute can exceed the per-interval budget.
3. 4xx errors other than 429, such as 400 content filter or invalid requests, 401/403 auth errors and 410 retired models.
4. Load spreading across deployments, multi-region back ends with circuit breakers, PTU plus spillover, queue-based load levelling, model fallback and graceful degradation (and client-side rate limiting).
5. The service estimates output tokens from the cap when accounting usage, so large caps consume rate-limit budget faster.
6. Retries multiply (client retries × gateway retries), increasing load during incidents and latency for users.

</details>

## References

- [Azure OpenAI quotas and limits](https://learn.microsoft.com/azure/foundry/openai/quotas-limits)
- [Manage Azure OpenAI quota](https://learn.microsoft.com/azure/foundry/openai/how-to/quota)
- [High availability and resiliency for Foundry](https://learn.microsoft.com/azure/foundry/how-to/high-availability-resiliency)
- [Use a gateway in front of multiple Azure OpenAI deployments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend)
- [Retry pattern (Architecture Center)](https://learn.microsoft.com/azure/architecture/patterns/retry)
- [Monitor Azure OpenAI](https://learn.microsoft.com/azure/foundry-classic/openai/how-to/monitor-openai)
