# Azure OpenAI in Foundry Models

## Why it matters

**Azure OpenAI in Foundry Models** (formerly Azure OpenAI Service) is the most widely used model family on Azure. It offers OpenAI's GPT, reasoning, image, video, audio, realtime and embedding models, sold by Microsoft with Azure's security, compliance, regional choices and enterprise terms. Knowing the families, how versions and the **v1 API** work, and how the Azure offering differs from OpenAI's own API prevents a lot of confusion when moving between documentation, SDKs and regions.

## Key concepts

### Model families (checked 2026-11)

| Family | Examples | Notes |
|---|---|---|
| **GPT-6** | `gpt-6-astra`, `gpt-6-sol`, `gpt-6-luna`, `gpt-6.1-sol`, `gpt-6.1-terra` | Reasoning, 1,050,000-token context (922,000 input, 128,000 output); Responses and Chat Completions; multi-agent orchestration (preview); computer use. Quota request may be needed depending on quota tier |
| **GPT-5.x** | `gpt-5.6-sol/terra/luna`, `gpt-5.5`, `gpt-5.4` (+ mini, nano, pro), `gpt-5.3-codex`, `gpt-5.2`, `gpt-5.1` (+ codex variants), `gpt-5`, `gpt-5-mini`, `gpt-5-nano`, `gpt-5-pro` | Reasoning models with `reasoning_effort` (values vary by model); codex variants for coding agents |
| **GPT-chat-latest** (Preview) | `gpt-chat-latest` | Rolling chat model |
| **o-series** | `o3`, `o3-pro`, `o4-mini`, `o3-mini`, `o1`, `codex-mini` | Earlier reasoning models |
| **GPT-4.1 / GPT-4o** | `gpt-4.1` (+ mini, nano), `gpt-4o`, `gpt-4o-mini` | Non-reasoning; older versions being retired (for example gpt-4o 2024-05-13 retires January 2027) |
| **Open-weight** | `gpt-oss-120b`, `gpt-oss-20b` | Apache 2.0 open-weight reasoning models, sold by Azure |
| **Embeddings** | `text-embedding-3-large`, `-3-small`, `ada-002` | File 09 |
| **Image** | `gpt-image-1`, `gpt-image-1-mini`, `gpt-image-1.5`, `gpt-image-2`, `gpt-image-2.5-*` | File 10 |
| **Video** | `sora-2` (Preview) | File 10 |
| **Audio and realtime** | `gpt-realtime` family, `gpt-audio` family, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize`, `gpt-4o-mini-tts`, `whisper`, `tts`, `tts-hd` | File 11 |
| **computer-use-preview** | | Responses API computer-use tool |

Names, versions and availability change monthly. The **models sold by Azure** page and the **region availability** page are the source of truth.

### Versions and deployments

- Each model has **versions** (for example `gpt-5-mini` version `2025-08-07`). A **deployment** pins a model and version (or follows auto-upgrade; file 14).
- You call a model by its **deployment name**, not the model name.
- A version's features can differ from OpenAI's API on the same day; check "API and feature support" tables.

### API surfaces

| API | Path (v1) | Use |
|---|---|---|
| **Responses** | `/openai/v1/responses` | Recommended for new work: stateful conversations, built-in tools, reasoning summaries, background mode (file 07) |
| **Chat Completions** | `/openai/v1/chat/completions` | Widely supported, stateless messages API |
| **Embeddings** | `/openai/v1/embeddings` | Vectors |
| **Images** | `/openai/v1/images/generations`, `/edits` | Image generation and editing |
| **Videos** | `/openai/v1/videos` (check current path) | Sora video jobs |
| **Audio** | `/openai/v1/audio/transcriptions`, `/translations`, `/speech` | Speech to text, text to speech |
| **Realtime** | WebRTC or WebSocket realtime endpoints | Low-latency speech and audio (file 11) |
| **Files, Batch, Fine-tuning** | `/openai/v1/files`, `/batches`, `/fine_tuning/jobs` | Batch processing and customisation |
| Assistants (Preview) | Legacy | Superseded by the Responses API and Foundry Agent Service |

### API versions and the v1 API

| | Date-versioned API (older) | **v1 API** (current) |
|---|---|---|
| URL | `https://<res>.openai.azure.com/openai/deployments/<deployment>/chat/completions?api-version=2024-10-21` | `https://<res>.openai.azure.com/openai/v1/chat/completions` |
| Versioning | Monthly `api-version` (GA and preview) required on every call | No `api-version` needed; new features arrive continuously |
| Client | `AzureOpenAI(api_version=..., azure_endpoint=...)` | Standard `OpenAI(base_url=".../openai/v1/", api_key=...)` |
| Deployment in URL | Yes | No; pass the deployment name as `model` |
| Status | Still supported for existing apps | Recommended for new code |

The v1 API makes code portable between OpenAI and Azure OpenAI (only `base_url` and authentication differ) and avoids frequent `api-version` updates.

### Azure OpenAI vs OpenAI's own API

| Aspect | Azure OpenAI in Foundry Models |
|---|---|
| Contract and terms | Microsoft; covered by Azure enterprise agreements and Product Terms |
| Data | Prompts and completions aren't used to train models; data at rest stays in the resource's geography; processing location set by deployment type |
| Identity and network | Microsoft Entra ID, managed identities, private endpoints, network isolation |
| Safety | Azure AI Content Safety filters and guardrails on by default; abuse monitoring (with modified abuse monitoring available for approved customers) |
| Availability | Model and feature rollout can lag or differ by region and deployment type |
| Integration | Foundry Agent Service, evaluations, Control Plane, AI gateway |

## How it works on Azure

### v1 API with keyless auth (Python)

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp)

resp = client.responses.create(
    model="gpt-5-mini",                      # deployment name
    input="Give me three test cases for a password reset API.",
    reasoning={"effort": "low"},
    max_output_tokens=800)
print(resp.output_text)
```

### Migrating from the date-versioned client

```python
# Before
from openai import AzureOpenAI
client = AzureOpenAI(azure_endpoint="https://<res>.openai.azure.com", api_version="2024-10-21",
                     azure_ad_token_provider=tp)
client.chat.completions.create(model="<deployment>", messages=[...])

# After (v1)
from openai import OpenAI
client = OpenAI(base_url="https://<res>.openai.azure.com/openai/v1/", api_key=tp)
client.chat.completions.create(model="<deployment>", messages=[...])
```

### Content filtering and guardrails

Every deployment has a content filter configuration (default or custom **RAI policy**): categories hate, sexual, violence and self-harm at severity thresholds, plus **Prompt Shields** (jailbreak and indirect attacks), protected material detection, and other guardrails. Filtered requests return HTTP 400 with `content_filter` details, and filtered completions return `finish_reason: "content_filter"`. Topic 23 covers configuration.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Which family | Start with a current mid-tier reasoning model (for example a GPT-5.x mini) for general tasks; move up for hard reasoning and agentic work, down (nano, 4.1-nano, SLMs) for high-volume simple tasks. Evaluate before choosing |
| Reasoning effort | Low or none for latency-sensitive tasks; medium or high for complex tasks; tune with evaluations |
| v1 vs date-versioned | v1 for new code; migrate legacy code opportunistically |
| Responses vs Chat Completions | Responses for new apps and agents; Chat Completions for compatibility |
| Version pinning | Pin versions in production with controlled upgrade windows (file 14) |

## Hands-on

1. Deploy `gpt-5-mini` and call it through the v1 endpoint with Entra ID using Responses and Chat Completions.
2. Rewrite a legacy `AzureOpenAI(api_version=...)` script to the v1 pattern.
3. Compare two model families on 20 of your own prompts for quality, latency and token usage.
4. Trigger a content filter (with a benign but borderline prompt) and inspect the error payload.
5. List your deployments' model versions and their upgrade settings.

## Security, cost and reliability notes

- **Security:** use Entra ID, private endpoints and disabled local auth (topic 14).
- **Security:** keep content filters on in production; modifications to filtering and abuse monitoring require approval and justification.
- **Cost:** newer models may have short-context and long-context pricing, reasoning tokens and cache-write charges; model the cost per request.
- **Reliability:** Global Standard can see latency variance at high volume; use provisioned or priority processing for strict latency (files 05, 12).

## Common mistakes

- Using the model name instead of the deployment name.
- Mixing date-versioned and v1 patterns inconsistently.
- Assuming a feature available in OpenAI's API is already available in Azure in your region.
- Choosing the largest model by default.
- Ignoring retirement dates of older GPT-4o and 4.1 versions.

## Check yourself

1. What's the difference between the date-versioned API and the v1 API?
2. Which client class do you use with the v1 API?
3. Name four Azure OpenAI model families available in Foundry.
4. Why might a feature be in OpenAI's API but not yet in Azure OpenAI?
5. What happens when a request is blocked by a content filter?
6. What value goes into `model` in an Azure OpenAI call?

<details>
<summary>Answers</summary>

1. The date-versioned API requires a monthly `api-version` parameter and deployment-scoped URLs; the v1 API is OpenAI-compatible at `/openai/v1/`, needs no `api-version`, and takes the deployment name as `model`.
2. The standard `OpenAI` client with `base_url` set to `https://<resource>.openai.azure.com/openai/v1/`.
3. Any four of: GPT-6, GPT-5.x, o-series, GPT-4.1/4o, gpt-oss, embeddings, gpt-image, sora-2, realtime/audio.
4. Azure rolls out features and models per region and deployment type and integrates them with Azure security and safety systems, so timing and availability can differ.
5. The request returns HTTP 400 with content filter details (or the completion ends with `finish_reason: content_filter`).
6. The deployment name.

</details>

## References

- [What is Azure OpenAI in Foundry Models?](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Foundry Models sold by Azure](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Azure OpenAI API version lifecycle (v1 API)](https://learn.microsoft.com/azure/foundry/openai/api-version-lifecycle)
- [Azure OpenAI reasoning models](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Guardrails and content filtering](https://learn.microsoft.com/azure/foundry/guardrails/guardrails-overview)
- [Data, privacy and security for Azure OpenAI](https://learn.microsoft.com/azure/foundry/responsible-ai/openai/data-privacy)
