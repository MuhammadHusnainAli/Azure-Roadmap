# Azure Translator

## Why it matters

Global organisations translate product content, support conversations, contracts, manuals and websites every day. **Azure Translator in Foundry Tools** powers translation across Microsoft products and offers real-time text translation in 100+ languages and dialects, document translation that preserves formatting, and customisation for domain terminology. In 2026 Translator gained an API version that lets you choose between classic **neural machine translation (NMT)** and **LLM-based translation** (for example GPT-5.1) with tone, gender and adaptive custom controls. Architects must pick the right engine per scenario for quality, consistency, cost and compliance.

## Key concepts

### Features

| Feature | What it does | Status / notes |
|---|---|---|
| **Text translation 2026-06-06** | Newest API: revised request/response schema (`inputs` / `value`), **model choice (NMT or LLM)**, adaptive custom translation, tone (formal/informal/neutral) and gender controls | GA; **breaking changes** from v3.0; LLM translation requires a Microsoft Foundry resource |
| **Text translation v3** | Real-time translation, transliteration, dictionary lookup, dynamic dictionary, no-translate markup | GA; SDKs and containers |
| **Document translation (asynchronous batch)** | Translate many or large files, preserving structure and format; source and target in Blob Storage | GA; SAS or managed identity for storage |
| **Document translation (synchronous)** | Translate a single file (optionally with a glossary) and get it back in the response; no Blob Storage needed | GA; custom domain endpoint required |
| **Adaptive custom translation** | Upload 5–10,000 pre-aligned segment pairs (≤500 characters each); the service builds a bilingual index in minutes to adapt supported LLMs | For LLM-based translation |
| **Custom Translator** | Train custom NMT models from parallel documents or dictionaries (web portal, no code) | Typical gains of 5–10 BLEU points with good data; to and from English |
| **Containers** | Text translation container | GA, **gated** (request access); connected or disconnected |

### NMT vs LLM translation

| | NMT | LLM-based |
|---|---|---|
| Strengths | Fast, consistent, cost-effective at high volume; mature customisation (Custom Translator, dictionaries) | Fluency, context awareness, tone and gender control, adaptive style from examples |
| Watch-outs | Less context across sentences; less stylistic control | Higher cost and latency, more variability; needs a Foundry resource and model deployment |
| Use for | UI strings, catalogues, high-volume support, regulated consistency | Marketing copy, customer-facing content needing tone, nuanced documents |

### Controlling terminology

| Mechanism | Use |
|---|---|
| **Dynamic dictionary** | Force a specific translation for a term within a request |
| **No-translate markup** | Keep product names or code untranslated |
| **Glossaries** (document translation) | Apply term lists across documents |
| **Custom Translator models** | Learn domain terminology and style from parallel data (NMT) |
| **Adaptive custom translation** | Reference translations or an adaptive index to steer LLM output |

## How it works on Azure

Text translation with the v3 SDK and keyless authentication:

```python
# pip install azure-ai-translation-text azure-identity
from azure.identity import DefaultAzureCredential
from azure.ai.translation.text import TextTranslationClient

client = TextTranslationClient(endpoint="https://contoso-foundry.cognitiveservices.azure.com/",
                               credential=DefaultAzureCredential())
result = client.translate(body=["Your order has shipped and will arrive tomorrow."],
                          to_language=["fr", "de", "ja"], from_language="en")
for item in result:
    for t in item.translations:
        print(t.to, t.text)
```

Batch document translation: upload source files to a Blob container, grant the Translator resource's **managed identity** access (Storage Blob Data Contributor), and submit a job specifying source, targets and optional glossaries; poll the job status.

The 2026-06-06 REST API (`api-version=2026-06-06`) is the path for LLM model selection and adaptive customisation; test thoroughly because request and response contracts changed.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **NMT vs LLM** | Default to NMT for volume and consistency; use LLM translation where tone and fluency matter and you've measured quality gains worth the cost. |
| **v3.0 vs 2026-06-06** | Stay on v3.0 until you've validated the new schema; migrate when you need LLM choice or adaptive features. |
| **Translator vs a general LLM prompt** | Translator offers document format preservation, glossaries, custom models, containers and predictable pricing; a raw LLM prompt is fine for low-volume ad hoc translation inside an agent. |
| **Custom Translator vs adaptive custom** | Custom Translator for NMT with substantial parallel data; adaptive custom translation for quickly steering LLM output with examples. |
| **Synchronous vs batch documents** | Synchronous for single files in interactive apps; batch for many or large files. |
| **Quality assurance** | Human review for high-stakes content (legal, medical, safety); automatic metrics (BLEU, COMET) for monitoring. |

## Hands-on

1. Translate 20 support replies into three languages with v3 NMT; time and cost them.
2. Add a dynamic dictionary entry so "Contoso Rewards" stays untranslated; verify.
3. Translate a formatted Word document with **synchronous document translation**; confirm formatting is preserved.
4. Submit a **batch** job for a folder of PDFs using managed identity to Blob Storage.
5. Call the 2026-06-06 API with an LLM model and a formal tone for the same marketing paragraph; compare with NMT output via native-speaker review or an LLM judge with a rubric.

## Security, cost and reliability notes

- **Keyless** with Entra and managed identity for storage; avoid SAS tokens where managed identity works.
- **No-trace**: Translator text translation doesn't persist submitted text for the standard service; custom models and documents in your storage follow your retention.
- **Sensitive content**: consider containers for documents that can't leave a site (gated access).
- **Cost**: billed per character (text) and per page or character (documents); LLM translation also incurs model token costs; custom model training and hosting extra.
- **Limits**: request size and character limits; batch large workloads.
- **Quality drift**: monitor terminology consistency, especially after switching engines or versions.

## Common mistakes

| Mistake | Fix |
|---|---|
| Migrating to 2026-06-06 without testing | Validate the new schema and output |
| LLM translation for millions of UI strings | NMT with dictionaries |
| No glossary for product names | Dynamic dictionary, no-translate markup, glossaries |
| Using SAS tokens with long expiry for document translation | Managed identity |
| No human review for legal or safety content | Add review workflow |

## Check yourself

1. What's new in Translator API version 2026-06-06?
2. What's needed to use LLM-based translation?
3. Compare synchronous and asynchronous document translation.
4. What is adaptive custom translation, and what data does it need?
5. When would you choose NMT over LLM translation?
6. Name three ways to control terminology.
7. How should document translation access Blob Storage securely?
8. What's special about the Translator container?

<details>
<summary>Answers</summary>

1. A revised request/response schema (breaking changes), choice of NMT or LLM model, adaptive custom translation, and tone and gender controls.
2. A Microsoft Foundry resource with a supported LLM deployment (for example GPT-5.1) and the 2026-06-06 API.
3. Synchronous translates one file and returns it in the response without Blob Storage; asynchronous batch translates many or large files between Blob containers.
4. A way to adapt LLM translation using 5–10,000 pre-aligned segment pairs (≤500 characters each) or reference translations, indexed in minutes.
5. For high-volume, consistency-critical, cost-sensitive translation such as UI strings and catalogues.
6. Any three: dynamic dictionary, no-translate markup, glossaries, Custom Translator models, adaptive custom translation.
7. With the Translator resource's managed identity granted Storage Blob roles (rather than SAS tokens).
8. It's GA but gated (requires access approval) and can run disconnected.

</details>

## References

- [What is Azure Translator?](https://learn.microsoft.com/azure/ai-services/translator/overview)
- [Text translation overview (2026-06-06)](https://learn.microsoft.com/azure/ai-services/translator/text-translation/overview)
- [Document translation overview](https://learn.microsoft.com/azure/ai-services/translator/document-translation/overview)
- [Custom Translator](https://learn.microsoft.com/azure/ai-services/translator/custom-translator/overview)
- [Foundry Tools containers](https://learn.microsoft.com/azure/ai-services/cognitive-services-container-support)
