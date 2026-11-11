# Foundry Tools Overview

## Why it matters

Before large language models, Microsoft's AI platform was a set of task-specific services: speech, vision, language, translation, document processing. They're now called **Foundry Tools** (formerly Azure AI services and, before that, Cognitive Services), and they remain essential. For many tasks (transcribing a call, extracting an invoice total, translating a manual, detecting PII, screening content) a purpose-built service is cheaper, faster, more accurate and easier to certify than prompting a general LLM. Agents increasingly use them as tools. Architects must know which services exist, how they are provisioned and secured, what runs in containers, and which are retiring.

## Key concepts

### The services

| Service | What it does | File |
|---|---|---|
| **Azure Vision** | Image analysis (captions, tags, objects, OCR), Face, multimodal embeddings | 02 |
| **Azure Speech** | Speech to text, text to speech, speech translation, speaker recognition, Voice Live | 03 |
| **Azure Language** | NER, PII detection, summarisation, conversational language understanding, question answering, sentiment | 04 |
| **Azure Translator** | Text and document translation, custom translator (100+ languages) | 05 |
| **Azure Document Intelligence** | Prebuilt, layout and custom document extraction | 06 |
| **Azure Content Understanding** | Multimodal extraction from documents, images, audio and video with analysers | 07 |
| **Azure AI Video Indexer** | Video and audio insights | 08 |
| **Azure AI Content Safety** | Harm detection, Prompt Shields, groundedness, protected material | 09 |
| **Azure AI Search** | Search and retrieval (covered in [Topic 17](../17-rag-and-knowledge/README.md)) | — |
| **Custom Vision**, **Immersive Reader** | Custom image classification/detection; reading assistance | 02, 10 |

**Retired or retiring** (don't use for new work): Anomaly Detector, Content Moderator (replaced by Content Safety), Language Understanding (LUIS, replaced by conversational language understanding), Metrics Advisor, Personalizer, QnA Maker (replaced by question answering). Some features inside services also retire (file 02 and file 04).

### Resource types

| Resource kind | Use |
|---|---|
| **Microsoft Foundry resource** (`kind: AIServices`) | Primary resource: one endpoint and identity boundary for Azure OpenAI, Foundry Models, agents and most Foundry Tools; projects for grouping work |
| **Single-service resources** (for example `SpeechServices`, `FormRecognizer`, `TextTranslation`, `ContentSafety`) | When you need a service-specific feature, region, pricing tier or isolation |
| **Free tier (F0)** | Many services offer F0 for trials, with low TPS limits |

Pricing is generally per transaction or unit (characters, audio hours, pages, images) by tier; tiers also set the maximum transactions per second.

### Authentication

| Method | Guidance |
|---|---|
| **Microsoft Entra ID** (managed identity, `DefaultAzureCredential`) | **Recommended**; requires a custom subdomain endpoint; assign roles such as Cognitive Services User |
| **API keys** (`Ocp-Apim-Subscription-Key`) | Shared secrets; disable with `disableLocalAuth` where possible; if used, store in Key Vault and rotate |
| **Tokens from keys** | Short-lived tokens for clients (for example Speech in browsers) |

### Containers

Many Foundry Tools run as **Docker containers** on-premises, at the edge or on AKS, for data locality, latency or disconnected scenarios. Containers report usage for billing (connected) or run **disconnected** with commitment-tier licensing (approval required).

| Area | Examples (status) |
|---|---|
| Language | Key phrases, language detection, sentiment, NER, PII, CLU (GA); summarisation (Preview) |
| Translator | Text translation (GA, gated) |
| Speech | Speech to text, custom speech to text, neural text to speech (GA); fast transcription (Preview) |
| Document Intelligence | Prebuilt and layout containers (see service docs) |
| Vision | Read OCR container **retiring 28 September 2028**; use Document Intelligence containers |
| Content Safety | Text, image, Prompt Shields containers (Preview) **retiring 30 April 2027** |

### Limited Access

Some capabilities need an application and approval (managed customers, approved use cases): Face identify/verify, Custom Neural Voice Pro, custom text-to-speech avatar, Speaker Recognition, Vision celebrity recognition, and Azure OpenAI modified content filters or abuse monitoring.

## How it works on Azure

| Surface | Use |
|---|---|
| **Foundry portal** | Try Speech, Vision, Language, Content Safety and others in playgrounds; manage from the same project as models and agents |
| **REST APIs and SDKs** | `azure-ai-vision-imageanalysis`, `azure-cognitiveservices-speech`, `azure-ai-textanalytics` / Language, `azure-ai-translation-text`, `azure-ai-documentintelligence`, `azure-ai-contentsafety` and others |
| **Agents** | Many services are callable as agent tools (Azure Speech tool, Content Understanding via Azure AI Search skills, OpenAPI/MCP wrappers) |
| **Azure AI Search skills** | Document Intelligence layout, Content Understanding, OCR, language skills in indexing pipelines |
| **Containers** | Docker/AKS deployments with billing endpoint configuration |

Networking: private endpoints and VNet rules on the resource; disable public network access for regulated workloads. Encryption at rest with Microsoft-managed or customer-managed keys.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Foundry resource vs single-service** | Default to the Foundry resource for simplicity and one governance boundary; use single-service resources when a feature, tier, region or isolation requirement needs it. |
| **Service vs LLM** | Use the purpose-built service for well-defined tasks with high volume, strict accuracy or compliance needs; LLMs for open-ended reasoning (file 11). |
| **Cloud vs container** | Containers for data that can't leave a site, low-latency edge, or disconnected operation; cloud for everything else (latest models, less operations). |
| **Keys vs Entra** | Entra with managed identity; keys only where Entra isn't supported. |
| **Region** | Feature and model availability differs by region (for example custom neural voice, layout features); check before deploying. |
| **Retirements** | Inventory usage of retired services and features; plan migrations (Content Moderator → Content Safety, LUIS → CLU, QnA Maker → question answering, Read container → Document Intelligence). |

## Hands-on

Create a Foundry resource with keyless access and call two tools.

```bash
az cognitiveservices account create -n contoso-foundry -g rg-ai -l swedencentral \
  --kind AIServices --sku S0 --custom-domain contoso-foundry --yes
az resource update --ids $(az cognitiveservices account show -n contoso-foundry -g rg-ai --query id -o tsv) \
  --set properties.disableLocalAuth=true
az role assignment create --assignee $(az ad signed-in-user show --query id -o tsv) \
  --role "Cognitive Services User" \
  --scope $(az cognitiveservices account show -n contoso-foundry -g rg-ai --query id -o tsv)
```

```python
# pip install azure-ai-textanalytics azure-ai-translation-text azure-identity
from azure.identity import DefaultAzureCredential
from azure.ai.textanalytics import TextAnalyticsClient

cred = DefaultAzureCredential()
lang = TextAnalyticsClient("https://contoso-foundry.cognitiveservices.azure.com/", cred)
docs = ["Call Jane Doe on +44 20 7946 0000 about order 4471."]
for d in lang.recognize_pii_entities(docs):
    print(d.redacted_text, [(e.text, e.category) for e in d.entities])
```

Then try the Speech and Vision playgrounds in the Foundry portal using the same resource.

## Security, cost and reliability notes

- **Keyless by default**; private endpoints for sensitive data; customer-managed keys where required.
- **Data handling**: most services process data transiently; custom models and stored data (for example custom voices, translation memories) persist in your resource. Review each service's data, privacy and security documentation.
- **Responsible AI**: transparency notes and Limited Access policies apply (Face, voice cloning).
- **Cost**: per-transaction pricing; commitment tiers for predictable high volume; F0 for trials only.
- **Throughput**: TPS limits per tier; handle 429s with retries and backoff; scale with multiple resources or containers.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building new solutions on retired services (LUIS, QnA Maker, Content Moderator) | Use their replacements |
| Using an LLM for high-volume OCR or PII detection | Use the purpose-built service |
| API keys in code | Entra ID and managed identity |
| Forgetting the custom subdomain needed for Entra auth | Create the resource with `--custom-domain` |
| Assuming every feature is in every region | Check regional availability |
| Ignoring Limited Access requirements | Apply early; plan alternatives |

## Check yourself

1. What were Foundry Tools previously called?
2. Name eight current Foundry Tools.
3. Which services are retired, and what replaces three of them?
4. What is the difference between the Foundry resource and single-service resources?
5. What is required to use Entra ID authentication with Foundry Tools?
6. Why use containers, and what's the difference between connected and disconnected?
7. Which capabilities are Limited Access?
8. Which two container offerings have announced retirement dates?

<details>
<summary>Answers</summary>

1. Azure AI services, and before that Cognitive Services.
2. Any eight: Vision, Speech, Language, Translator, Document Intelligence, Content Understanding, Video Indexer, Content Safety, Azure AI Search, Custom Vision, Immersive Reader.
3. Anomaly Detector, Content Moderator, LUIS, Metrics Advisor, Personalizer, QnA Maker; Content Moderator → Content Safety, LUIS → conversational language understanding, QnA Maker → question answering.
4. The Foundry resource (`AIServices`) is one endpoint and governance boundary for models, agents and most tools; single-service resources expose one service, sometimes needed for specific features, tiers or isolation.
5. A custom subdomain endpoint and an Entra role assignment (for example Cognitive Services User).
6. Data locality, latency, edge and offline needs; connected containers send billing data to Azure, disconnected ones run offline under approved commitment licensing.
7. Face identify/verify, Custom Neural Voice Pro, text-to-speech avatar, Speaker Recognition, celebrity recognition, and Azure OpenAI modified content filters/abuse monitoring.
8. Vision Read OCR container (28 September 2028) and Content Safety containers (30 April 2027).

</details>

## References

- [What are Foundry Tools?](https://learn.microsoft.com/azure/ai-services/what-are-ai-services)
- [Set up your first Foundry resource](https://learn.microsoft.com/azure/ai-services/multi-service-resource)
- [Authenticate requests to Foundry Tools](https://learn.microsoft.com/azure/ai-services/authentication)
- [Foundry Tools containers](https://learn.microsoft.com/azure/ai-services/cognitive-services-container-support)
- [Limited Access features](https://learn.microsoft.com/azure/ai-services/cognitive-services-limited-access)
