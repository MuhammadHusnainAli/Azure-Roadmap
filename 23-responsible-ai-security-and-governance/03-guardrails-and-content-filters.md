# Guardrails and Content Filters

## Why it matters

Base models are safety-trained, but that training can be bypassed, and it knows nothing about your policies, your users or your tools. **Guardrails** are the runtime safety system that sits around models and agents in Microsoft Foundry: classifiers that check prompts, tool calls, tool responses and outputs for harmful content, prompt attacks, protected material and personal data, and then annotate or block. They are the "Protect" layer you configure, version in IaC and test with evaluations and red teaming. The design questions are which risks, at which intervention points, at which severity, with what action, and what the user sees when something is blocked.

## Key concepts

### Vocabulary

| Term | Meaning |
|---|---|
| **Guardrail** | A named collection of controls, applied to one or more model deployments and/or agents (ARM resource: **RAI policy**) |
| **Control** | Risk + intervention point(s) + action (+ blocking level for harm categories) |
| **Intervention point** | Where content is scanned: user input, tool call (Preview), tool response (Preview), output |
| **Action** | Annotate (models only) or annotate and block |
| **Default guardrail** | `Microsoft.DefaultV2` assigned to models by default |

Agent guardrails are **Preview**. Guardrails apply to all Foundry Models sold by Azure except audio transcription models.

### Risks

| Risk | Models | Agents (Preview) | Notes |
|---|---|---|---|
| Hate, Sexual, Self-harm, Violence | ✅ | ✅ | Severity-based blocking |
| User prompt attacks (jailbreak) | ✅ | ✅ | Prompt Shields; on by default |
| Indirect attacks (document attacks) | ✅ | ✅ | Prompt Shields; off by default; scan documents and tool responses |
| Spotlighting (Preview) | ✅ | ❌ | Marks untrusted document content to reduce indirect injection |
| Protected material for text and code | ✅ | ✅ | Code annotations include example source repository and licence |
| Groundedness (Preview) | ✅ | ❌ | **Retires 30 April 2027** in Azure OpenAI; use evaluators or your own grounding checks |
| PII (Preview) | ✅ | ✅ | Detect, filter or redact personal data |
| Task adherence (Preview) | ✅ | ✅ | Flags tool use misaligned with the user's task |

### Blocking levels for harm categories

| Level | Blocks | Availability |
|---|---|---|
| Off | Nothing (annotations may still return) | Inputs: all customers. Outputs: approved customers only (modified guardrails) |
| Low | High severity only | All |
| Medium | Medium and high | All (typical default) |
| High | Low, medium and high | All |

### Intervention points

| Point | Applies to | Example control |
|---|---|---|
| **User input** | Models, agents | User prompt attack → annotate and block |
| **Tool call** (Preview) | Agents | Scan arguments sent to a tool for harmful content or PII |
| **Tool response** (Preview) | Agents | Indirect attack in tool output → agent stops |
| **Output** | Models, agents | Protected material text → annotate |

Tool-call and tool-response scanning needs moderation support from the tool: Azure AI Search, Azure Functions, OpenAPI, SharePoint grounding, Fabric data agent, Bing grounding, Bing Custom Search and Browser Automation. Other tools aren't scanned at those points. Each intervention point adds roughly **50–100 ms** of latency.

### Inheritance

- A model deployment uses its assigned guardrail (default `Microsoft.DefaultV2`).
- An agent with a custom guardrail uses **only** that guardrail; it fully overrides the model's guardrail.
- An agent without a custom guardrail inherits its model deployment's guardrail.

So an agent guardrail weaker than the model's guardrail weakens protection; review agent guardrails as carefully as model ones.

### Other building blocks

| Feature | Use |
|---|---|
| **Custom blocklists** | Exact terms or regex patterns to block in inputs or outputs (competitor names, internal code names, profanity lists) |
| **Network egress controls** (Preview) | For hosted agents (and prompt agents using the Copilot harness): restrict outbound destinations, configured in the same guardrail |
| **Safety system messages** | Instruction templates (harmful content, grounding, jailbreak, copyright) that complement classifiers |
| **Streaming modes** | Default (buffered, filtered before release) or **asynchronous filter** (lower latency, content may be shown before filtering; requires approval) |
| **Azure AI Content Safety APIs** | Use the same classifiers in non-Foundry apps or custom pipelines ([Topic 20 file 09](../20-foundry-tools/09-azure-ai-content-safety.md)) |

## How it works on Azure

### Portal

**Guardrails → Create**: add controls (risk, intervention points, action, level), assign to agents and models, name and create. Test in the playground with a non-production deployment, since assignment takes effect immediately.

### Bicep (RAI policy) and assignment

```bicep
resource foundry 'Microsoft.CognitiveServices/accounts@2025-06-01' existing = { name: 'aif-contoso-prod' }

resource guardrail 'Microsoft.CognitiveServices/accounts/raiPolicies@2024-10-01' = {
  parent: foundry
  name: 'gr-customer-support'
  properties: {
    basePolicyName: 'Microsoft.DefaultV2'
    mode: 'Default'
    // Filter names are strings; export a portal-created policy (az resource show) to confirm exact names.
    contentFilters: [
      { name: 'Hate',      blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Prompt' }
      { name: 'Hate',      blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Completion' }
      { name: 'Violence',  blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Prompt' }
      { name: 'Violence',  blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Completion' }
      { name: 'Sexual',    blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Prompt' }
      { name: 'Sexual',    blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Completion' }
      { name: 'Selfharm',  blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Prompt' }
      { name: 'Selfharm',  blocking: true, enabled: true, severityThreshold: 'Medium', source: 'Completion' }
      { name: 'Jailbreak', blocking: true, enabled: true, source: 'Prompt' }
      { name: 'Indirect Attack', blocking: true, enabled: true, source: 'Prompt' }
      { name: 'Protected Material Text', blocking: true, enabled: true, source: 'Completion' }
    ]
  }
}

resource deployment 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundry
  name: 'gpt-5-mini'
  sku: { name: 'GlobalStandard', capacity: 100 }
  properties: {
    model: { format: 'OpenAI', name: 'gpt-5-mini', version: '2025-08-07' }
    raiPolicyName: guardrail.name
  }
}
```

The ARM schema's newer preview API versions add agent intervention points as `source` values (`PreRun`, `PreToolCall`, `PostToolCall`, `PostRun`) alongside `Prompt` and `Completion`; check the current template reference before using them. Agent guardrail assignment is done in the portal or agent APIs.

### Handling filtered requests

| Case | Response | App behaviour |
|---|---|---|
| Prompt blocked | HTTP 400 with `content_filter` error code and category details | Show a neutral message; log the category, not the content |
| Output blocked | `finish_reason: content_filter` | Show partial or fallback response; offer escalation |
| Annotation only | `prompt_filter_results` / `content_filter_results` with `detected` and `filtered` | Use for monitoring, UI warnings, or custom logic |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Baseline** | Start from `Microsoft.DefaultV2`; tighten for children, public or vulnerable audiences. |
| **Indirect attacks** | Enable for every RAG app and every agent with retrieval or tools; add tool-response scanning for supported tools. |
| **Agent guardrails** | Give each production agent an explicit guardrail at least as strict as its model's. |
| **PII** | Block or redact PII in outputs for public apps; consider tool-call scanning so agents don't send PII to external tools. |
| **Blocklists** | For business-specific terms; keep them small and reviewed. |
| **Latency budget** | Each intervention point adds ~50–100 ms; prioritise the highest-risk points. |
| **Modified guardrails** | Turning off output filters requires Limited Access approval; justify and document. |
| **Defence in depth** | Guardrails are probabilistic; combine with least privilege, approvals and egress control (file 04). |

## Hands-on

1. Create a guardrail with Medium harm thresholds, jailbreak and indirect attack blocking and protected material; assign it to a test deployment.
2. Send benign, borderline and attack prompts in the playground; inspect annotations.
3. Create an agent guardrail with indirect attack scanning at tool response; give the agent an Azure AI Search tool with a document containing injected instructions; confirm the run stops.
4. Add a blocklist with two internal code names; test.
5. Deploy the same guardrail with Bicep and run your safety evaluation suite ([Topic 22 file 03](../22-evaluation-observability-and-genaiops/03-built-in-evaluators.md)) before and after.

## Security, cost and reliability notes

- **Not a complete defence**: classifiers miss some attacks (and are primarily trained on English for prompt attacks); combine with architectural controls.
- **Agent override risk**: an agent guardrail replaces the model's; audit for weaker agent guardrails with policy and reviews.
- **Change control**: guardrails are production configuration; version in IaC and review changes.
- **Monitoring**: track block rates by category; spikes indicate attacks or a misconfiguration hurting users.
- **Groundedness detection retirement** (30 April 2027): migrate to evaluators or custom grounding validation.

## Common mistakes

| Mistake | Fix |
|---|---|
| Leaving document attack detection off for RAG | Enable indirect attacks |
| Agent guardrail weaker than model guardrail | Explicit, reviewed agent guardrails |
| Expecting tool-response scanning on unsupported tools | Check supported tools; add your own validation |
| Showing raw filter errors to users | Friendly fallback and escalation path |
| Logging blocked content verbatim | Log category and severity; restrict access |

## Check yourself

1. Define guardrail, control, intervention point and action.
2. Which intervention points are agent-only?
3. How do agent and model guardrails interact?
4. What do the Low, Medium and High blocking levels block?
5. Which risks apply to models but not agents?
6. How do you assign a guardrail to a deployment in Bicep?

<details>
<summary>Answers</summary>

1. Guardrail: named collection of controls assigned to models/agents. Control: risk, intervention point(s), action and level. Intervention point: where content is scanned. Action: annotate, or annotate and block.
2. Tool call and tool response (both Preview).
3. An agent's custom guardrail fully overrides the model's; without one, the agent inherits the model deployment's guardrail.
4. Low: high severity only; Medium: medium and high; High: low, medium and high.
5. Spotlighting and groundedness (both Preview).
6. Create a `Microsoft.CognitiveServices/accounts/raiPolicies` resource and set `raiPolicyName` on the deployment's properties.

</details>

## References

- [Guardrails and controls overview](https://learn.microsoft.com/azure/foundry/guardrails/guardrails-overview)
- [Create guardrails](https://learn.microsoft.com/azure/foundry/guardrails/how-to-create-guardrails)
- [Intervention points](https://learn.microsoft.com/azure/foundry/guardrails/intervention-points)
- [Default safety policies](https://learn.microsoft.com/azure/foundry/openai/concepts/default-safety-policies)
- [Content filter severity levels](https://learn.microsoft.com/azure/foundry/openai/concepts/content-filter-severity-levels)
- [Use blocklists](https://learn.microsoft.com/azure/foundry/openai/how-to/use-blocklists)
- [raiPolicies template reference](https://learn.microsoft.com/azure/templates/microsoft.cognitiveservices/accounts/raipolicies)
