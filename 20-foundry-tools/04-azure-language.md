# Azure Language

## Why it matters

Text is everywhere: support tickets, chat logs, call transcripts, contracts, clinical notes. **Azure Language in Foundry Tools** provides natural language processing APIs for detecting and redacting personal data, recognising entities, detecting language, analysing health text and (in legacy features) classifying intents, answering questions, summarising and scoring sentiment. In the LLM era, Microsoft has split the service into **core capabilities** (actively invested, recommended for new work) and **legacy capabilities** (supported for existing implementations). Knowing which is which, and when a task-specific API beats an LLM, avoids building on features that are winding down.

## Key concepts

### Core capabilities (recommended for new development)

| Capability | What it does |
|---|---|
| **PII detection** | Identifies and redacts personal data in **text**, **conversations** (chat, transcripts) and **documents**; anonymisation by synthetic replacement is in Preview |
| **Language detection** | Detects language and dialect variants |
| **Prebuilt NER** | Recognises entities (person, organisation, location, date, quantity and more) |
| **Custom NER** | Train models to extract your own entity types from unstructured text |
| **Text analytics for health** | Extracts and labels medical entities, relations and assertions from clinical text |

### Legacy capabilities (supported for existing implementations)

| Capability | What it does | Typical modern alternative for new work |
|---|---|---|
| **Conversational language understanding (CLU)** | Intents and entities from utterances (successor to LUIS) | LLM with structured outputs, or CLU where deterministic intent models are required |
| **Custom text classification** | Single/multi-label document classification | LLM classification or fine-tuned small model ([Topic 16](../16-prompt-engineering-and-model-customisation/README.md)) |
| **Entity linking** | Links entities to Wikipedia | LLM or knowledge graph |
| **Key phrase extraction** | Main concepts | LLM extraction |
| **Orchestration workflow** | Routes between CLU and question answering projects | Agent routing |
| **Question answering** | Curated FAQ answers (successor to QnA Maker) | RAG / Foundry IQ, or the Exact Question Answering agent |
| **Sentiment analysis and opinion mining** | Sentiment and aspect-level opinions | LLM with rubric, or keep for cheap high-volume scoring |
| **Summarisation** | Extractive and abstractive text, conversation and call-centre summarisation | LLM summarisation |

"Legacy" doesn't mean retired: these features remain supported for existing workloads, but new investment goes to the core features and to LLM-based approaches.

### Agent-oriented tools

| Tool | Purpose |
|---|---|
| **Azure Language MCP server** | Remote (via the Foundry tool catalogue) or local MCP server exposing Language capabilities to agents |
| **Intent Routing agent** | Prebuilt agent for predictable intent-based routing with controlled responses |
| **Exact Question Answering agent** | Word-for-word answers to critical business questions with human oversight |

### Deployment

Cloud API (Foundry resource or Language resource), Foundry portal playgrounds (including a GA Document PII playground), and **containers** (key phrases, language detection, sentiment, NER, PII and CLU GA; summarisation Preview) with connected or disconnected operation.

## How it works on Azure

Redact PII with keyless authentication (Python, `azure-ai-textanalytics`):

```python
from azure.identity import DefaultAzureCredential
from azure.ai.textanalytics import TextAnalyticsClient

client = TextAnalyticsClient(endpoint="https://contoso-foundry.cognitiveservices.azure.com/",
                             credential=DefaultAzureCredential())
docs = ["Customer Jane Doe (jane@contoso.com, +44 20 7946 0000) disputes invoice 77812."]
for doc in client.recognize_pii_entities(docs, language="en"):
    print(doc.redacted_text)
    for e in doc.entities:
        print(f"  {e.category:<20} {e.text:<25} {e.confidence_score:.2f}")
```

You can filter categories (`categories_filter`) and choose domains (for example PHI for health). Conversation PII works on transcripts with speaker turns; document PII on files.

Language in RAG pipelines: Azure AI Search includes PII detection, entity recognition and language detection **skills** for enrichment ([Topic 17 file 03](../17-rag-and-knowledge/03-ingestion-indexers-and-skillsets.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Language API vs LLM** | Use Language for PII redaction, NER and language detection at high volume with deterministic, auditable output and predictable cost; use LLMs for nuanced classification, summarisation and extraction with flexible schemas (file 11). |
| **Redact before LLMs** | Run PII detection before sending text to LLMs, logs or training datasets where data minimisation applies. |
| **Custom NER vs LLM extraction** | Custom NER for stable, high-volume entity types with labelled data; LLM extraction for evolving schemas and low volume. |
| **Legacy features** | Keep existing CLU, question answering or sentiment if they work; for new projects evaluate core features and LLM approaches first. |
| **Containers** | For text that must not leave the premises (for example PII detection on-site before upload). |
| **Health text** | Text analytics for health has domain-specific models and compliance documentation; review responsible AI and regulatory guidance. |

## Hands-on

1. Run the PII sample above on 20 support tickets; review false positives and negatives by category.
2. Add a `categories_filter` so only emails, phone numbers and credit card numbers are redacted; compare.
3. Redact a call transcript with **conversation PII** and pass the redacted text to an LLM for summarisation.
4. Compare sentiment analysis (legacy API) with an LLM scoring rubric on 50 reviews: agreement, cost per 1,000 documents and latency.
5. Connect the **Azure Language MCP server** from the Foundry tool catalogue to an agent and ask it to detect the language and entities in a message.

## Security, cost and reliability notes

- **PII detection isn't perfect**: measure recall on your data; combine with other controls (access restrictions, retention).
- **Keyless access**; private endpoints when processing sensitive text.
- **Data processing**: text is processed transiently for prebuilt features; custom model training data is stored in your resource/storage.
- **Cost**: billed per text record (1,000 characters) by feature; containers and commitment tiers for scale.
- **Limits**: document size and documents-per-request limits; batch requests and handle 429s.
- **Health and regulated data**: follow responsible AI notes and your compliance obligations.

## Common mistakes

| Mistake | Fix |
|---|---|
| Starting new chatbots on CLU + question answering by default | Evaluate RAG and agents first |
| Sending raw customer text to LLMs and logs | Redact PII first |
| Treating PII detection as a guarantee | Measure and layer controls |
| Using LLMs for bulk language detection | Use the Language API |
| Confusing "legacy" with "retired" | Legacy is supported; LUIS and QnA Maker are retired |

## Check yourself

1. Which Azure Language features are core, and which are legacy?
2. What three input types does PII detection support?
3. What replaced LUIS and QnA Maker?
4. When should you prefer the Language API over an LLM?
5. Why redact PII before LLM calls?
6. What agent-oriented tools does Azure Language offer?
7. Which Language containers are GA?
8. How is Language priced?

<details>
<summary>Answers</summary>

1. Core: PII detection, language detection, prebuilt NER, custom NER, text analytics for health. Legacy: CLU, custom text classification, entity linking, key phrases, orchestration workflow, question answering, sentiment/opinion mining, summarisation.
2. Text, conversations (chat and transcripts) and documents.
3. Conversational language understanding (LUIS) and question answering (QnA Maker).
4. High-volume, well-defined tasks needing deterministic, auditable output and predictable cost (PII redaction, NER, language detection).
5. Data minimisation: avoid exposing personal data to models, logs, caches and training sets unnecessarily.
6. Azure Language MCP server (remote and local), Intent Routing agent, Exact Question Answering agent.
7. Key phrase extraction, language detection, sentiment, NER, PII and CLU (summarisation is Preview).
8. Per text record (units of characters) per feature, with free and standard tiers and commitment options.

</details>

## References

- [What is Azure Language in Foundry Tools?](https://learn.microsoft.com/azure/ai-services/language-service/overview)
- [PII detection quickstart](https://learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/quickstart)
- [Conversational language understanding](https://learn.microsoft.com/azure/ai-services/language-service/conversational-language-understanding/overview)
- [Text analytics for health](https://learn.microsoft.com/azure/ai-services/language-service/text-analytics-for-health/overview)
- [Foundry Tools containers](https://learn.microsoft.com/azure/ai-services/cognitive-services-container-support)
