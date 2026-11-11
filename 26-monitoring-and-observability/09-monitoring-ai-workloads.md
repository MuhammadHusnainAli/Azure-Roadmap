# Monitoring AI Workloads

## Why it matters

AI workloads fail in ways ordinary dashboards don't show. A model deployment can be "up" while every request is throttled with HTTP 429; latency can double because prompts grew; costs can triple because an agent loops through tools; a provisioned deployment can saturate and spill over; content filters can start blocking legitimate users after a prompt change. Operating AI in production needs the usual golden signals plus **token usage**, **time to first token**, **throttling and quota**, **PTU utilisation**, **cache hit rates**, **content filter events** and **agent traces**. Quality and safety monitoring is covered in [Topic 22 file 07](../22-evaluation-observability-and-genaiops/07-continuous-evaluation-and-monitoring.md); this file focuses on the operational telemetry.

## Key concepts

### Signal map

| Signal | Source | Why |
|---|---|---|
| **Requests and errors** (by status code, deployment) | Platform metric `AzureOpenAIRequests` (dimensions: ModelDeploymentName, ModelName, StatusCode, IsSpillover, StreamType, ServiceTier) | Availability, 429s, 5xx |
| **Tokens** | `ProcessedPromptTokens` (input), `GeneratedTokens` (output), token transaction metrics | Cost, capacity, prompt growth |
| **Latency** | `AzureOpenAINormalizedTTFTInMS` (time to first byte/token), `AzureOpenAITimeToResponse`, `AzureOpenAITTLTInMS` (time to last byte), normalised time between tokens | User experience for streaming and non-streaming |
| **PTU utilisation** | `AzureOpenAIProvisionedManagedUtilizationV2` | Capacity planning; above 100% causes 429s or spillover |
| **Prompt cache** | `AzureOpenAIContextTokensCacheMatchRate` | Cost and latency savings from caching |
| **Content filter events** | Guardrail annotations in responses; resource logs; Defender for AI alerts | Blocked requests, attack attempts ([Topic 23](../23-responsible-ai-security-and-governance/README.md)) |
| **Gateway telemetry** | API Management `llm-emit-token-metric`, logs | Tokens per app/team, quota enforcement ([Topic 11 file 07](../11-integration-messaging-and-apis/07-api-management-as-ai-gateway.md)) |
| **Agent traces** | Foundry tracing (OpenTelemetry GenAI spans) in Application Insights | Tool calls, steps, errors, per-run tokens ([Topic 22 file 06](../22-evaluation-observability-and-genaiops/06-tracing-with-opentelemetry.md)) |
| **Dependencies** | Azure AI Search, Cosmos DB, Storage metrics | Retrieval latency and throttling |

Microsoft's tip: always pair a latency metric with a token metric; latency that rises with token counts is expected, latency rising without more tokens indicates a real issue.

### Throttling and spillover

- **429 Too Many Requests** when exceeding tokens-per-minute (TPM)/requests-per-minute quota or PTU capacity; responses include retry-after headers.
- **Spillover** routes excess provisioned traffic to a standard deployment; track `IsSpillover`.
- Monitor 429 rate per deployment and client; alert before users notice.

### Where data lives

| Data | Store |
|---|---|
| Platform metrics | Azure Monitor Metrics (automatic) |
| Resource logs (requests, audit) | Log Analytics via diagnostic settings |
| Traces, app telemetry, agent spans | Application Insights (Log Analytics) |
| Gateway logs/metrics | APIM diagnostics and Application Insights |
| Foundry dashboards | Agent Monitoring Dashboard, Foundry Control Plane |

## How it works on Azure

Metric alerts to create for every production deployment:

| Alert | Condition |
|---|---|
| Throttling | `AzureOpenAIRequests` where StatusCode = 429 > 2% of requests over 5 minutes |
| Server errors | StatusCode 5xx > 1% |
| Latency | `AzureOpenAINormalizedTTFTInMS` p95 (or average) above SLO for 15 minutes (dynamic threshold) |
| PTU saturation | `AzureOpenAIProvisionedManagedUtilizationV2` > 90% for 15 minutes |
| Token spike | `GeneratedTokens` total > dynamic threshold (cost anomaly) |

Application Insights query: tokens and latency per agent and model from GenAI spans (attribute names follow the OpenTelemetry GenAI conventions, which are still evolving; verify in your data):

```kusto
AppDependencies
| where TimeGenerated > ago(24h)
| where tostring(Properties["gen_ai.operation.name"]) in ("chat", "invoke_agent")
| extend model = tostring(Properties["gen_ai.response.model"]),
         agent = tostring(Properties["gen_ai.agent.name"]),
         inTok = toint(Properties["gen_ai.usage.input_tokens"]),
         outTok = toint(Properties["gen_ai.usage.output_tokens"])
| summarize calls = count(), p95_ms = percentile(DurationMs, 95), tokens = sum(inTok + outTok),
            failures = countif(Success == false) by agent, model
| order by tokens desc
```

Gateway token metrics by consumer (APIM emits custom metrics with dimensions you choose, for example subscription ID or calling app), enabling chargeback and per-team quotas.

Dashboard layout for an AI service: requests/429s/5xx by deployment; TTFT and time-to-last-byte with token counts; PTU utilisation; cache match rate; tokens per app (gateway); agent runs, tool failures and tokens per run; content filter block rate; cost estimate.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Gateway everywhere** | Route AI traffic through APIM for consistent per-consumer metrics, limits and logs. |
| **Content logging** | Don't log prompts/completions by default in production; use governed trace content recording when needed. |
| **SLOs** | Define SLOs for TTFT and success rate per user journey, not per model. |
| **Capacity alerts** | Alert on PTU utilisation and 429 trends early; pair with quota and failover plans (Topic 28 file 06). |
| **Cost monitoring** | Daily token totals per app with anomaly alerts (Topic 29 file 06). |
| **Agent metrics** | Track tokens and tool calls per run; set max-turn limits to stop runaway costs. |

## Hands-on

1. Enable diagnostic settings for a Foundry resource; create the five alerts in the table.
2. Build a workbook showing requests by status code, TTFT with token counts and PTU utilisation.
3. Put APIM in front of the model with token metric emission per subscription; chart tokens per consumer.
4. Run an agent with tracing; run the GenAI KQL query; find the most expensive agent.
5. Load-test until 429s appear; confirm alerts fire and retry-after handling works.

## Security, cost and reliability notes

- **Privacy**: telemetry may contain prompts; keep content recording off unless governed.
- **Security signals**: content filter blocks and jailbreak alerts should reach security (Defender for AI).
- **Reliability**: retries with backoff respecting retry-after; circuit breakers to alternative deployments.
- **Cost**: monitoring itself costs (logs, traces); sample traces for high-volume apps.

## Common mistakes

| Mistake | Fix |
|---|---|
| Monitoring only HTTP success | Track 429s, tokens, TTFT, PTU |
| Latency alerts without token context | Pair latency with token metrics |
| No per-consumer visibility | APIM token metrics |
| Discovering cost spikes on the invoice | Token anomaly alerts |
| Logging full prompts by default | Governed content recording |

## Check yourself

1. Which platform metrics track requests, tokens, latency and PTU utilisation?
2. Why pair latency with token metrics?
3. What causes 429s, and how do you detect spillover?
4. How do you get per-team token usage?
5. Where do agent traces come from?
6. List five alerts for a production model deployment.

<details>
<summary>Answers</summary>

1. `AzureOpenAIRequests`; `ProcessedPromptTokens` and `GeneratedTokens`; `AzureOpenAINormalizedTTFTInMS`, `AzureOpenAITimeToResponse`, `AzureOpenAITTLTInMS`; `AzureOpenAIProvisionedManagedUtilizationV2`.
2. Longer outputs naturally take longer; latency rising without token growth signals a real problem.
3. Exceeding TPM/RPM quota or PTU capacity; split `AzureOpenAIRequests` by `IsSpillover`.
4. API Management token metrics with consumer dimensions.
5. Foundry tracing and OpenTelemetry GenAI spans in Application Insights.
6. 429 rate, 5xx rate, TTFT above SLO, PTU utilisation > 90%, token spike/anomaly.

</details>

## References

- [Monitoring data reference for Azure OpenAI](https://learn.microsoft.com/azure/foundry/openai/monitor-openai-reference)
- [Monitor agents with the Agent Monitoring Dashboard](https://learn.microsoft.com/azure/foundry/observability/how-to/how-to-monitor-agents-dashboard)
- [Set up tracing in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/observability/how-to/trace-agent-setup)
- [API Management GenAI gateway capabilities](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
