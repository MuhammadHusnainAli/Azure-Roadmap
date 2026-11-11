# Data Privacy, Residency and Abuse Monitoring

## Why it matters

The first question every legal, privacy or procurement team asks about generative AI is "what happens to our data?" Does OpenAI see it? Is it used for training? Where is it processed? Is anything stored, and who can read it? Answering precisely, from Microsoft's documented commitments rather than assumptions, is a core skill for architects and consultants. The details depend on choices you make: deployment type (Global, Data Zone, regional), which features store state (Responses API, agents, files, fine-tuning, traces), whether abuse monitoring is modified, and how your own app logs prompts. This file covers the commitments, the data flows and the design levers.

## Key concepts

### Microsoft's commitments for Models sold by Azure

For Foundry **Models sold by Azure** (including Azure OpenAI), your prompts, completions, embeddings and training data:

- are **not** available to other customers;
- are **not** available to OpenAI or other model providers;
- are **not** used by model providers to improve their models or services;
- are **not** used to train any generative AI foundation models without your permission or instruction;
- are **not** used to improve Microsoft or third-party products without your explicit permission.

Fine-tuned models are exclusively yours. Models are hosted in Azure and don't interact with provider services such as ChatGPT or the OpenAI API. Models are **stateless**: nothing is stored in the model itself.

Models **not** sold by Azure (for example, some partner models via Marketplace) have their own terms; check each model's licence and data terms.

### What data is processed

| Data | Stored? | Where / notes |
|---|---|---|
| Prompts and completions (inference) | Not by the model | Evaluated synchronously by guardrails; may be sampled for abuse monitoring |
| Uploaded files, vector stores | Yes | In the Foundry resource, same geography, encrypted (Microsoft-managed key, optional CMK) |
| Stateful features: Responses API history, agent conversations, threads | Yes | Service-managed storage, encrypted at rest; deletable |
| Batch input | Yes, until processed | Batch is a Global deployment type |
| Fine-tuning data and models | Yes | Exclusive to you; deletable |
| "On your data" retrieval sources | No duplicate store | Data stays in your source |
| **Your** logs, traces, evaluation results | Yes, under your control | Application Insights, storage, Cosmos DB; your retention and access controls |

Use `store=false` on the Responses API for stateless interactions where you don't want service-managed history (reasoning models then return encrypted reasoning items for you to pass back).

### Residency and deployment types

| Deployment type | Where prompts and responses may be processed |
|---|---|
| **Standard / Provisioned (regional)** | Within the resource's geography (may move between regions within the geography for operations) |
| **Data Zone** (for example EU, US) | Any region within the data zone |
| **Global** | Any geography where the model is deployed |

Data **at rest** (files, stored state, fine-tuning data) stays in the resource's geography regardless of deployment type. For EU customers, combine Data Zone EU deployments with EU-region resources and check the EU Data Boundary documentation for each service ([Topic 15 file 04](../15-foundry-models-and-azure-openai/04-deployment-types.md)).

### Abuse monitoring

| Component | What happens |
|---|---|
| **Content classification** | Classifiers detect harmful content in prompts and completions |
| **Abuse pattern capture** | Algorithms score usage patterns (frequency and severity of harmful content, recurring jailbreak attempts) |
| **Review and decision** | Flagged content may be reviewed by automated systems (including LLMs) and, where confidence is insufficient, by authorised Microsoft employees |
| **Notification and action** | Customer notified by email; usually an opportunity to remediate except for severe or recurring abuse |

Content stored for human review is logically separated by customer resource. Automated review doesn't store prompts and completions or use them for training.

**Modified abuse monitoring**: customers processing highly sensitive data can apply (Limited Access) to turn off data storage and human review; automated review may still run. Verify with the resource's `ContentLogging` capability, which appears as `false` only when storage is off:

```bash
az cognitiveservices account show -n aif-contoso-prod -g rg-ai-prod --query "properties.capabilities[?name=='ContentLogging']"
```

### "Zero data retention" in practice

There is no single switch. Achieving minimal retention means combining:

1. Approved modified abuse monitoring (no storage for human review).
2. Stateless APIs (`store=false`) and no service-side threads, files or memory, or deleting them promptly.
3. No content recording in traces (Topic 22 file 06), short Application Insights retention.
4. Your app not logging prompts and responses, or logging them only redacted.
5. Purview retention policies aligned with the minimum required (file 05).

## How it works on Azure

Privacy design checklist for an AI workload:

| Area | Decision |
|---|---|
| **Data classification** | What categories of personal or confidential data enter prompts, retrieval and tools? |
| **Lawful basis and notices** | Update privacy notices; inform users about AI processing, retention and human review |
| **DPIA** | Data protection impact assessment for high-risk processing (GDPR Article 35) |
| **Residency** | Deployment type and region per data classification; document it |
| **Storage** | Which features persist state; retention and deletion procedures; CMK where required |
| **Abuse monitoring** | Default or modified; justification |
| **Telemetry** | Content recording off in production; trace retention; access via Log Analytics roles |
| **Subject rights** | How to find and delete a user's data across conversations, memory, traces, evaluation datasets |
| **Network** | Private endpoints; no public access for sensitive workloads |
| **Contracts** | Microsoft Products and Services Data Protection Addendum (DPA) and Product Terms |

Azure Policy can enforce allowed regions and deployment types (for example deny Global deployments for restricted subscriptions) ([Topic 03 file 04](../03-governance-and-landing-zones/04-ai-landing-zones.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Global vs Data Zone vs regional** | Global for best availability and price; Data Zone when processing must stay in a zone; regional/provisioned for strict geography or capacity guarantees. |
| **Stateful APIs** | Convenient for multi-turn and agents; stateless with your own encrypted store when you need full control. |
| **Modified abuse monitoring** | Apply for highly sensitive data; be ready to respond to abuse notifications without Microsoft's human review context. |
| **CMK** | When policy requires customer control of keys; check feature coverage (some preview features don't support CMK). |
| **Partner models** | Review each model's terms; prefer Models sold by Azure for regulated data. |

## Hands-on

1. List every place your agent's data is stored (service state, files, memory, traces, evaluation data, your logs) with retention and deletion method.
2. Configure a Data Zone EU deployment and an Azure Policy denying Global deployments in a test subscription.
3. Call the Responses API with `store=false` and confirm no stored response is retrievable.
4. Check the `ContentLogging` capability on a resource.
5. Draft the AI section of a privacy notice and a DPIA outline.

## Security, cost and reliability notes

- **Your own logs are usually the biggest risk**, not the model provider: govern traces and app logs.
- **Preview features** may have different privacy practices and not support CMK; check before using regulated data.
- **Global deployments** offer higher quota and resilience; restricting to regional may cost more or need provisioned throughput.
- **Abuse notifications**: have a contact and process to respond quickly to avoid service interruption.

## Common mistakes

| Mistake | Fix |
|---|---|
| "Azure OpenAI sends data to OpenAI" | It doesn't; cite the documented commitments |
| Assuming Global deployments store data globally | Processing location varies; data at rest stays in geography |
| Claiming "zero retention" with default settings | Combine modified abuse monitoring, stateless APIs and your own log controls |
| Forgetting traces and evaluation datasets in subject-rights processes | Include all stores |
| Using partner models without reading their terms | Review per-model data terms |

## Check yourself

1. List Microsoft's five commitments about prompts and completions for Models sold by Azure.
2. Which features store data in the service?
3. Compare processing locations for regional, Data Zone and Global deployments.
4. Describe the four components of abuse monitoring.
5. How do you verify that abuse monitoring data storage is off?
6. What steps together minimise data retention?

<details>
<summary>Answers</summary>

1. Not available to other customers; not available to OpenAI or other providers; not used by providers to improve models; not used to train foundation models without permission; not used to improve Microsoft or third-party products without explicit permission.
2. Files and vector stores, Responses API history, agent conversations/threads, stored completions, batch input, fine-tuning data and models.
3. Regional: within the resource's geography. Data Zone: any region in the zone. Global: any geography where the model is deployed. Data at rest stays in the resource's geography.
4. Content classification, abuse pattern capture, review and decision (automated, then human if needed), notification and action.
5. `az cognitiveservices account show` and look for the `ContentLogging` capability set to `false` (it only appears when off).
6. Modified abuse monitoring, stateless APIs or prompt deletion of stored state, no content recording in traces, minimal app logging with redaction, and aligned retention policies.

</details>

## References

- [Data, privacy and security for Models sold by Azure](https://learn.microsoft.com/azure/foundry/responsible-ai/openai/data-privacy)
- [Abuse monitoring](https://learn.microsoft.com/azure/foundry/openai/concepts/abuse-monitoring)
- [Deployment types](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)
- [Tracing and data handling](https://learn.microsoft.com/azure/foundry/observability/concepts/trace-data)
