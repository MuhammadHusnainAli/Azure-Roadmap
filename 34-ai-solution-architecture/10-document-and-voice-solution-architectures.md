# Document and Voice Solution Architectures

## Why it matters

Two of the highest-ROI AI patterns in enterprises are **intelligent document processing (IDP)**, turning invoices, claims, contracts, forms and correspondence into structured data and decisions, and **voice and contact centre automation**, answering calls with natural speech, resolving routine requests and assisting human agents. Both combine multiple services (document extraction, LLMs, workflows, speech, telephony, CRM) and have demanding non-functional requirements: accuracy and human review for documents; sub-second latency, interruption handling and seamless human handoff for voice. This file presents reference designs using Azure Document Intelligence, Content Understanding, Foundry models and agents, Durable Functions, Azure AI Speech Voice Live, Azure Communication Services and Dynamics 365 Contact Center.

## Key concepts

### Intelligent document processing

| Stage | Purpose | Azure services |
|---|---|---|
| **Ingest** | Receive documents from email, upload, scanners, SFTP, APIs | Logic Apps, Event Grid, Blob Storage, Service Bus |
| **Classify** | Determine document type | Content Understanding classifiers, Document Intelligence custom classification, LLM classification |
| **Extract** | Fields, tables, line items, signatures | **Document Intelligence** prebuilt (invoice, receipt, ID, tax forms) and custom models; **Content Understanding** analysers (schemas, multimodal, generative extraction with confidence) ([Topic 20 files 06–07](../20-foundry-tools/06-azure-document-intelligence.md)) |
| **Validate and enrich** | Business rules, lookups (supplier master, policy), LLM reasoning for ambiguous fields | Functions, Foundry models/agents, databases |
| **Human review** | Low-confidence or high-value items reviewed and corrected | Review UI (Power Apps, custom app), queue with SLAs |
| **Integrate** | Post to ERP/claims/CRM; archive | Logic Apps connectors, APIM, SAP/Dynamics integration (file 11) |
| **Learn** | Feed corrections back to improve models | Retraining data, evaluation sets |
| **Orchestrate** | Long-running, resumable workflow | **Durable Functions**, Logic Apps, Agent Framework workflows |

Key design points: confidence thresholds per field drive straight-through processing vs review; idempotent processing; audit trail of extracted values and corrections; PII handling and retention.

### Voice and contact centre

| Component | Purpose | Azure services |
|---|---|---|
| **Telephony** | PSTN numbers, call routing, SIP, recording | **Azure Communication Services** (Call Automation, direct routing), Teams Phone, Dynamics 365 Contact Center |
| **Real-time speech** | Speech-to-speech conversation with low latency, barge-in, noise suppression | **Azure AI Speech Voice Live API**, real-time audio models (`gpt-realtime` family) via Foundry ([Topic 18 file 11](../18-agents-with-foundry-agent-service/11-voice-agents.md)) |
| **Agent logic** | Intent handling, tools (account lookup, booking), knowledge | Foundry Agent Service or custom orchestrator with tools |
| **Handoff** | Transfer to human with context | Call Automation transfer, Dynamics 365 Contact Center/Copilot Studio Customer Conversation harness |
| **Agent assist** | Real-time transcription, suggested answers, summaries for human agents | Speech, Foundry models, Dynamics 365 Contact Center Copilot |
| **Post-call analytics** | Transcripts, sentiment, compliance, QA | Speech batch transcription, Language, Fabric |

Voice NFRs: end-to-end latency (aim for well under a second to first audio), interruption handling, accents and languages, authentication (voice or knowledge-based), PCI DSS for card payments (pause recording, DTMF masking), consent and call recording disclosures.

### Build vs buy

| Option | Fit |
|---|---|
| **Dynamics 365 Contact Center** | Full contact centre with Copilot and voice agents; buy |
| **Copilot Studio (Customer Conversation harness)** | Low-code customer conversation agents with escalation (Topic 24 file 03) |
| **Custom on ACS + Voice Live + Foundry** | Full control, custom UX and integrations |

## How it works on Azure

IDP for supplier invoices:

```text
Email → Logic Apps → Blob (raw) → Event Grid → Durable Functions orchestrator
  1. Content Understanding / Document Intelligence invoice model → fields + confidence
  2. Validate: supplier in master data (SQL), PO match (ERP API), totals check; LLM resolves ambiguous line descriptions
  3. Route: all fields ≥ 0.9 confidence and rules pass → post to ERP (APIM) ; else → review queue (Power Apps)
  4. Corrections stored → evaluation/training set; audit record in Cosmos DB
Metrics: straight-through rate, accuracy per field, cycle time, cost per document
```

Voice agent for appointment booking:

```text
PSTN call → ACS phone number → Call Automation → media streaming to app (Container Apps)
→ Voice Live API (speech in/out, barge-in) ↔ agent logic (Foundry agent with tools: find_slots, book, verify_identity)
→ on low confidence or request: transfer to human queue with transcript summary
→ post-call: transcript to Storage, summary and sentiment to CRM, QA metrics in Fabric
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Document Intelligence vs Content Understanding vs LLM-only** | Prebuilt/custom models for standard forms and high volume; Content Understanding for multimodal, schema-driven generative extraction; LLM-only for low-volume, varied documents with validation. |
| **Straight-through threshold** | Set per field and value; review high-value items regardless of confidence. |
| **Voice pipeline** | Speech-to-speech (Voice Live/realtime) for natural conversation; cascaded STT → LLM → TTS for more control and lower cost. |
| **Handoff design** | Make escalation easy and context-rich; measure containment with CSAT. |
| **Compliance** | PCI for payments (avoid capturing card data in AI transcripts), recording consent, retention. |

## Hands-on

1. Build an invoice extraction flow with Document Intelligence and a confidence-based review queue.
2. Compare Content Understanding and an LLM-only extractor on 50 varied documents; measure accuracy and cost.
3. Build a voice agent with Voice Live and one tool; measure latency and handle barge-in.
4. Add human handoff with a context summary.
5. Define KPIs: straight-through rate, containment, CSAT, cost per document/call.

## Security, cost and reliability notes

- **PII and payment data**: redact transcripts and extracted data where required; restrict access; retention policies.
- **Cost**: per-page extraction, per-minute speech/real-time audio, telephony minutes, model tokens; model per document/call.
- **Reliability**: queue-based processing for documents; graceful fallback to humans for voice outages.
- **Safety**: guardrails for voice agents; disclose AI use to callers.

## Common mistakes

| Mistake | Fix |
|---|---|
| Fully automating without confidence thresholds | Human review for low confidence/high value |
| No feedback loop from corrections | Capture and evaluate |
| Voice agent latency over several seconds | Speech-to-speech, streaming, fewer tool hops |
| Hard-to-reach human agent | Clear escalation paths |
| Card numbers in transcripts | PCI-compliant capture and redaction |

## Check yourself

1. List the stages of an IDP pipeline and the Azure services for each.
2. How do confidence thresholds drive automation?
3. Which components make up a voice agent architecture on Azure?
4. Compare speech-to-speech and cascaded voice pipelines.
5. When would you buy Dynamics 365 Contact Center instead of building?
6. Which KPIs measure IDP and voice solutions?

<details>
<summary>Answers</summary>

1. Ingest (Logic Apps/Event Grid/Blob), classify (Content Understanding/Document Intelligence/LLM), extract (Document Intelligence/Content Understanding), validate (Functions/models), human review (Power Apps/custom UI), integrate (Logic Apps/APIM), learn (evaluation data), orchestrate (Durable Functions).
2. High-confidence, rule-passing items go straight through; low-confidence or high-value items go to human review.
3. Telephony (ACS/Teams/Contact Center), real-time speech (Voice Live/realtime models), agent logic with tools (Foundry), handoff, agent assist and post-call analytics.
4. Speech-to-speech: more natural and lower latency; cascaded: more control, flexible model choice, often cheaper.
5. When you need a full contact centre with routing, workforce features and Copilot capabilities without custom build.
6. IDP: straight-through rate, field accuracy, cycle time, cost per document. Voice: containment, CSAT, average handle time, transfer rate, cost per call.

</details>

## References

- [Automate document processing (Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/automate-document-processing-azure-form-recognizer)
- [Document classification with Durable Functions](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/automate-document-classification-durable-functions)
- [Content Understanding overview](https://learn.microsoft.com/azure/ai-services/content-understanding/overview)
- [Voice Live API](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
- [Call Automation](https://learn.microsoft.com/azure/communication-services/concepts/call-automation/call-automation)
- [Dynamics 365 Contact Center](https://learn.microsoft.com/dynamics365/contact-center/implement/overview-contact-center)
