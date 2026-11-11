# Foundry Tools vs LLM Decision Guide

## Why it matters

Since multimodal LLMs arrived, almost any task Foundry Tools perform (OCR, translation, transcription, entity extraction, classification, moderation) can be attempted with a prompt. That doesn't mean it should be. Task-specific services are often **cheaper, faster, more accurate, more deterministic and easier to certify**; LLMs are more **flexible**, handle **ambiguity and reasoning**, and need **no training**. The best production systems combine them: a service does the heavy, well-defined work and an LLM reasons over its output. This guide gives a repeatable way to decide, and is the kind of judgement interviewers and clients test.

## Key concepts

### The core trade-off

| Dimension | Foundry Tools (task-specific) | LLMs (general-purpose) |
|---|---|---|
| **Accuracy on the defined task** | High and measured (OCR, layout, ASR, translation, PII) | Variable; strong on reasoning, weaker on precise localisation and long-tail OCR |
| **Determinism** | Stable outputs for the same input | Non-deterministic; can hallucinate |
| **Structured output** | Typed fields, confidence scores, bounding boxes, timestamps | Structured outputs possible; confidence and grounding need extra work |
| **Cost at volume** | Per page/minute/character; predictable; commitment tiers | Per token; images and audio consume many tokens; reasoning tokens add up |
| **Latency** | Optimised, often low | Higher, especially with reasoning |
| **Flexibility** | Fixed capabilities; customisation via training | Any task describable in language; zero-shot |
| **New schema or task** | Needs a new model or service feature | Change the prompt |
| **Compliance and audit** | Documented models, transparency notes, containers, disconnected options | Model lifecycle changes; harder to certify |
| **Edge / offline** | Containers (many GA) | Small models via Foundry Local; larger models need cloud |
| **Lifecycle** | Some services retiring (Image Analysis, Custom Vision) | Models retire on a schedule ([Topic 15 file 14](../15-foundry-models-and-azure-openai/14-model-lifecycle.md)) |

### Task-by-task guidance

| Task | Default choice | When the alternative wins |
|---|---|---|
| OCR, layout, tables from documents | **Document Intelligence** | LLM for light ad hoc reading inside an agent |
| Invoices, receipts, IDs, tax forms | **Document Intelligence prebuilt** | Content Understanding for highly variable or inferred fields |
| Unstructured or multimodal extraction (contracts, mixed media) | **Content Understanding** | Custom LLM pipeline for niche, fully controlled workflows |
| Speech transcription (calls, captions) | **Azure Speech** (batch, fast, real-time) | LLM Speech / realtime models for conversational agents and contextual transcription |
| Text translation at scale | **Translator NMT** | Translator LLM mode or an LLM for tone-sensitive marketing content |
| PII detection and redaction | **Azure Language PII** | LLM for context-dependent sensitive info, with Language as a first pass |
| Named entities, language detection | **Azure Language** | LLM for custom, evolving entity types at low volume |
| Sentiment, key phrases, summarisation | **LLM** for new work (these Language features are legacy) | Language API for cheap high-volume scoring in existing pipelines |
| Intent classification for routing | **LLM with structured outputs** | CLU where deterministic, auditable intent models are required |
| Image captioning, tagging, visual Q&A | **Multimodal LLM** (Image Analysis is retiring) | Content Understanding for managed, schema-driven image extraction |
| Custom object detection with bounding boxes | **Azure ML AutoML / detection models** | LLMs aren't reliable at precise localisation |
| Face verification | **Face API** (Limited Access) | — (LLMs must not be used for biometric identification) |
| Video indexing, captions, search | **Video Indexer** or Content Understanding video | LLM for short-clip reasoning |
| Content moderation and prompt attack detection | **Content Safety** | LLM-as-judge for nuanced policy evaluation, alongside Content Safety |

### Combination patterns

| Pattern | Example |
|---|---|
| **Service → LLM** | Document Intelligence Layout → Markdown → LLM answers questions or checks policy compliance |
| **LLM → service** | LLM drafts a reply → Content Safety screens it → Translator localises it |
| **Service as agent tool** | Agent calls Speech, Language MCP server or Document Intelligence via tools |
| **Pre-filter + LLM** | Language PII redaction before sending text to an LLM |
| **Cascade** | Cheap service or small model first; escalate uncertain cases (low confidence) to an LLM or a human |

## How it works on Azure

Everything lives in the same **Microsoft Foundry resource**: model deployments and most Foundry Tools share one endpoint, identity and governance boundary, so combining them doesn't multiply security work. Content Understanding even orchestrates your Foundry model deployments for you. Agents reach services as tools (built-in, OpenAPI, MCP such as the Azure Language MCP server).

## Design decisions and trade-offs

### Decision checklist

1. **Is the task well defined and supported by a service?** If yes, start with the service.
2. **Volume and cost**: estimate monthly cost both ways (per page/minute vs tokens, including image tokens and retries).
3. **Accuracy**: build a labelled test set; compare service vs LLM vs combination.
4. **Determinism and audit**: do regulators or auditors need stable, explainable outputs and confidence scores?
5. **Latency**: interactive or batch?
6. **Data location**: must processing happen on-premises or offline (containers)?
7. **Change rate**: will the schema or task change often? Favours LLMs.
8. **Lifecycle**: is the service or model retiring?

### Rules of thumb

- High-volume, well-defined perception tasks (read, hear, translate, detect) → **service**.
- Reasoning, synthesis, open-ended extraction, conversation → **LLM**.
- Regulated decisions → service outputs with confidence, LLM explanation, **human review**.
- Prototype with an LLM; productionise the high-volume parts with services if cost or accuracy demands.

## Hands-on

Run a head-to-head on 50 real documents (for example supplier invoices):

| Option | Setup |
|---|---|
| A | Document Intelligence `prebuilt-invoice` |
| B | Content Understanding custom analyser |
| C | Multimodal LLM with structured outputs on page images |
| D | Document Intelligence Layout Markdown → LLM structured extraction |

Measure field-level accuracy against labels, latency per document, cost per 1,000 documents, and percentage needing human review (low confidence). Present a one-page recommendation with the trade-offs. Repeat a smaller version for call transcription (Speech batch vs LLM audio transcription) or translation (NMT vs LLM).

## Security, cost and reliability notes

- **Data minimisation**: pre-process with services (redaction, extraction) so less raw data reaches LLMs.
- **Prompt injection** affects LLM paths (text inside documents and images); services aren't instruction-following, but their outputs fed to LLMs still need delimiting.
- **Cost surprises**: images and audio are token-hungry for LLMs; reasoning effort multiplies output tokens; compare at production volume.
- **Reliability**: services offer stable outputs and containers; LLM outputs need validation (schemas, confidence, evaluation).
- **Retirements**: plan replacements for retiring services and models in the same roadmap.

## Common mistakes

| Mistake | Fix |
|---|---|
| "The LLM can do it all" for high-volume OCR or transcription | Use services for perception at scale |
| Building new solutions on retiring services (Image Analysis, Custom Vision) | Use successors |
| Comparing on a few demos | Use labelled test sets and cost models |
| Ignoring confidence scores and human review | Design review paths for low confidence |
| Treating legacy Language features as the default for new work | Evaluate LLM approaches first |
| Using LLMs for biometric identification | Never; use governed Face API with Limited Access |

## Check yourself

1. List five dimensions on which Foundry Tools and LLMs differ.
2. Which choice would you default to for invoices, call transcription, and image captioning, and why?
3. Why aren't LLMs suitable for precise object detection?
4. Describe three combination patterns.
5. What should a decision checklist include?
6. When is an LLM better than the Translator NMT engine?
7. How would you prove your choice to a client?
8. Why pre-process with Azure Language before LLM calls?

<details>
<summary>Answers</summary>

1. Any five: accuracy on defined tasks, determinism, structured output with confidence, cost at volume, latency, flexibility, compliance/audit, edge/offline support, lifecycle.
2. Invoices: Document Intelligence prebuilt (accurate typed fields, confidence, cost). Call transcription: Azure Speech (accuracy, diarisation, cost at scale). Image captioning: multimodal LLM (Image Analysis is retiring and LLMs describe images flexibly).
3. They don't reliably produce precise bounding boxes and localisation; detection models (Azure ML AutoML) do.
4. Service → LLM (layout then reasoning), LLM → service (draft then safety screening and translation), service as an agent tool, pre-filter + LLM, cascade with escalation.
5. Task definition and service support, volume and cost, accuracy on a test set, determinism and audit needs, latency, data location, change rate, lifecycle.
6. For content where tone, fluency and context matter (marketing, nuanced documents), after measuring the quality gain against cost.
7. A head-to-head on labelled real data measuring accuracy, latency, cost per 1,000 items and human-review rate.
8. To detect and redact PII (data minimisation) and to reduce what sensitive data reaches models, logs and caches.

</details>

## References

- [What are Foundry Tools?](https://learn.microsoft.com/azure/ai-services/what-are-ai-services)
- [Choose the right Foundry Tool for document processing](https://learn.microsoft.com/azure/ai-services/content-understanding/choosing-right-ai-tool)
- [Migrate from Image Analysis](https://learn.microsoft.com/azure/ai-services/computer-vision/migration-options)
- [What is Azure Language? (core vs legacy features)](https://learn.microsoft.com/azure/ai-services/language-service/overview)
- [Text translation overview (NMT and LLM)](https://learn.microsoft.com/azure/ai-services/translator/text-translation/overview)
- [What is Azure Speech?](https://learn.microsoft.com/azure/ai-services/speech-service/overview)
