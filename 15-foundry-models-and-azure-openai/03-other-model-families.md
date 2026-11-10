# Other Model Families

## Why it matters

Foundry isn't OpenAI-only. Many teams get better quality, cost, latency, licensing or regional fit from other families: Anthropic Claude for coding and long-context agentic work, Microsoft's MAI and Phi models, Meta Llama, Mistral, xAI Grok, DeepSeek, Cohere for retrieval, Moonshot Kimi, Black Forest Labs for images, and models served by Fireworks AI or NVIDIA. A multi-model strategy reduces vendor risk and lets you route each task to the best-fit model. It also brings different APIs, terms, lifecycles and safety behaviour, which you need to manage.

## Key concepts

### Families at a glance (checked 2026-11)

| Provider | Examples in Foundry | Category | Notes |
|---|---|---|---|
| **Anthropic** | `claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`, `claude-opus-5`, `claude-sonnet-5`, `claude-opus-4-8/4-7/4-6/4-5`, `claude-sonnet-4-6/4-5`, `claude-haiku-4-5`, `claude-fable-5-x`; Mythos models are a gated research preview | Partner (Marketplace) | Messages API; up to 1,000,000-token context on newer models; hosted on Azure (some, Data Zone US) or on Anthropic infrastructure; strong coding and agentic tool use; some sampling parameters unsupported |
| **Microsoft MAI** | `MAI-Thinking-1` (Preview, reasoning chat), `MAI-Image-2.5` / `2.6` family (Preview, image generation and editing), MAI-Voice and MAI-Transcribe (through Azure Speech) | Sold by Azure | Microsoft's in-house frontier models |
| **Microsoft other** | `model-router`, `Microsoft-Decision-1` (calibrated decision scoring), healthcare AI models | Sold by Azure | Router covered in file 06 |
| **Microsoft Phi** | `Phi-4`, `Phi-4-mini-instruct`, `Phi-4-multimodal-instruct`, `Phi-4-reasoning`, `Phi-4-mini-reasoning` | Partners and community listing | Small, MIT-licensed open models; also on Foundry Local (topic 13 file 06) |
| **Meta** | `Llama-3.3-70B-Instruct`, `Llama-4-Maverick-17B-128E-Instruct-FP8` (sold by Azure); `Llama-4-Scout-17B-16E-Instruct` (partner) | Both | Llama community licence |
| **Mistral AI** | `Mistral-Large-3` (Preview), `mistral-medium-3-5` (Preview), `mistral-document-ai-2512`, `mistral-ocr-4-0` (Preview) sold by Azure; Codestral, Ministral, Mistral small/medium as partners | Both | Strong European option; OCR and document AI models |
| **xAI** (listed as SpaceXAI) | `grok-4.7` (GA, 500,000-token context, Chat Completions and Responses), `grok-4.6`, `grok-4.3`, `grok-4-20-reasoning/non-reasoning`, `grok-4.1-fast-*`, `grok-4`, `grok-code-fast-1` | Sold by Azure | Registration required for some models; GA Grok models from 4.7 have a minimum six-month availability |
| **DeepSeek** | `DeepSeek-V4-Pro`, `DeepSeek-V4-Flash`, `DeepSeek-V4.1-Flash`, `DeepSeek-V3.2`, `DeepSeek-V3.2-Speciale` | Sold by Azure | Reasoning and MoE models; 12-month GA lifecycle |
| **Cohere** | `Cohere-command-a`, `Cohere-command-a-plus-05-2026`, `Cohere-embed-v5-Pro/Fast`, `Cohere-rerank-v5-pro/fast`, `Cohere-rerank-v4.0-*`, `Cohere-parse-v5`; `embed-v-4-0`; embed v3 (partner) | Both | Strong retrieval stack: embeddings and **rerankers** for RAG (topic 17) |
| **Moonshot AI** | `Kimi-K3.1-Pro` (1,000,000-token input), `Kimi-K2.5/2.6/2.7-Code` (Preview) | Sold by Azure | Multimodal reasoning, English and Chinese |
| **Black Forest Labs** | `FLUX.2-pro`, `FLUX.2-flex`, `FLUX.1-Kontext-pro`, `FLUX-1.1-pro` | Sold by Azure | Image generation and editing |
| **OpenAI open weights** | `gpt-oss-120b`, `gpt-oss-20b` | Sold by Azure | Apache 2.0 |
| **Fireworks AI** | Models served through Fireworks, such as GLM and Kimi variants (for example `FW-GLM-5.3`, `FW-Kimi-K3` in the model router pool) | Partner | Fast inference for open models; 12-month GA lifecycle |
| **NVIDIA** | NVIDIA NIM microservices and NVIDIA-optimised models in the catalogue (managed compute) | Partner and community | GPU-optimised serving of open models (check current listings) |

The list grows constantly. Treat it as a map of who's there, not a fixed inventory.

### API differences across families

| Family | Primary API in Foundry | Watch out for |
|---|---|---|
| Azure OpenAI | OpenAI v1 (Responses, Chat Completions) | Reasoning parameter rules |
| Claude | **Anthropic Messages API** (`/anthropic` endpoint, Anthropic SDK); also usable in Agent Service and model router | Different message and tool schema; `thinking`, `effort` and caching semantics; some models reject `temperature` and `top_k`, and `top_p` is fixed |
| Most models sold by Azure (Grok, DeepSeek, Llama, Mistral, Kimi, MAI-Thinking) | Chat Completions-compatible on the Foundry endpoint (some also support Responses) | Tool calling and structured output support vary by model |
| Image models (FLUX, MAI-Image) | Model-specific image endpoints | Different parameters (size limits, editing inputs) |
| Rerankers (Cohere) | Rerank API | Not a chat model |

### Lifecycle differences

| Provider | GA lifecycle (Microsoft policy, 2026) |
|---|---|
| Azure OpenAI and most models sold by Azure | 18 months from launch (deprecated at 12 months for new customers) |
| Anthropic, DeepSeek, Fireworks, Mistral AI | **12 months** |
| xAI Grok (from 4.7) | Minimum **six months** |

Plan evaluations and upgrades accordingly (file 14).

## How it works on Azure

### Calling Claude with the Anthropic SDK (keyless)

```python
# pip install -U anthropic azure-identity
from anthropic import AnthropicFoundry
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

# Note the Foundry scope for the Anthropic endpoint
tp = get_bearer_token_provider(DefaultAzureCredential(), "https://ai.azure.com/.default")
client = AnthropicFoundry(azure_ad_token_provider=tp,
                          base_url="https://<resource>.services.ai.azure.com/anthropic")

msg = client.messages.create(
    model="<claude-deployment-name>",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Review this function for bugs: def add(a,b): return a-b"}])
print(msg.content)
```

The Entra ID scope for the Anthropic endpoint is `https://ai.azure.com/.default`; the OpenAI v1 endpoint uses `https://cognitiveservices.azure.com/.default`. The JavaScript equivalent uses the `@anthropic-ai/foundry-sdk` package. Some Claude models (for example the Mythos models) support Entra ID authentication only. If your subscription has zero data retention enabled, Anthropic-designated "covered models" may reject requests; see the Claude troubleshooting table.

### Calling a non-OpenAI model through Chat Completions

```python
from openai import OpenAI
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp)   # tp from above
r = client.chat.completions.create(model="<grok-or-deepseek-deployment>",
                                   messages=[{"role": "user", "content": "Summarise the EU AI Act in 3 bullets."}])
```

Many models sold by Azure are reachable through the same v1 endpoint with Chat Completions. Check each model's "API endpoints" section for the supported APIs.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Multi-model vs single provider | Multi-model for resilience, best-fit routing and negotiating power; single provider for simplicity. Many organisations standardise on two or three families plus the model router |
| Coding and agentic work | Evaluate Claude, GPT-5.x/6 codex and reasoning variants, Grok code and Kimi code models on your own repository tasks |
| Retrieval quality | Consider Cohere embed and rerank models alongside OpenAI embeddings and the AI Search semantic ranker |
| Regulated data | Prefer models sold by Azure, or partner models hosted on Azure, in the right data zone; review provider terms |
| Abstraction | Use the AI gateway's unified model API or the Agent Framework to keep app code independent of provider-specific APIs |
| Safety | Run your safety evaluations per model; refusal and filtering behaviour varies |

## Hands-on

1. Deploy three models from different providers (for example a GPT-5.x mini, a Claude Sonnet and a DeepSeek or Grok model).
2. Run the same 30-prompt evaluation set on each (quality, latency, tokens, cost) using Foundry evaluations.
3. Call Claude through the Anthropic SDK and the others through Chat Completions; note the API differences.
4. Add a Cohere reranker to a small retrieval experiment and compare precision@5 with and without it.
5. Document the lifecycle (retirement dates) of each deployed model from the retirement schedule.

## Security, cost and reliability notes

- **Security:** confirm data processing location and provider data-use terms per model, especially for partner models hosted outside Azure.
- **Security:** content filter coverage and Prompt Shields support can vary by model; verify before production.
- **Cost:** compare total cost per task (tokens × price, plus reasoning tokens and retries), not list price per token.
- **Reliability:** shorter lifecycles for some partners mean more frequent upgrades. Automate evaluations so upgrades are cheap.

## Common mistakes

- Assuming all models share OpenAI's API, parameters and tool-calling behaviour.
- Ignoring Marketplace terms for partner models.
- Picking a model from a benchmark without testing on your workload.
- Forgetting that some models (for example Mythos) are gated or preview-only.
- Hard-coding provider-specific APIs throughout an application.

## Check yourself

1. Which API do you use for Claude models in Foundry?
2. Name two Microsoft MAI models and their purpose.
3. Which providers have a 12-month GA lifecycle?
4. Why might you add Cohere models to a RAG system?
5. How can you keep application code independent of model providers?
6. What's special about how Grok models are listed?

<details>
<summary>Answers</summary>

1. The Anthropic Messages API (Anthropic SDK against the Foundry `/anthropic` endpoint).
2. MAI-Thinking-1 (reasoning chat, preview) and MAI-Image models (image generation and editing, preview); MAI-Voice and MAI-Transcribe are available through Azure Speech.
3. Anthropic, DeepSeek, Fireworks and Mistral AI.
4. For high-quality multilingual embeddings and rerankers that improve retrieval precision.
5. Use an abstraction layer such as the API Management unified model API, the model router, or a framework such as the Microsoft Agent Framework, and keep provider choice in configuration.
6. They're listed under SpaceXAI on the "models sold by Azure" page; some need registration, and GA models from Grok 4.7 have a minimum six-month availability.

</details>

## References

- [Foundry Models sold by Azure](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Foundry Models from partners and community](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-from-partners)
- [Deploy and use Claude models in Foundry](https://learn.microsoft.com/azure/foundry/foundry-models/how-to/use-foundry-models-claude)
- [Deploy and use Grok models in Foundry](https://learn.microsoft.com/azure/foundry/foundry-models/how-to/use-foundry-models-grok)
- [Use MAI-Thinking models](https://learn.microsoft.com/azure/foundry/foundry-models/how-to/use-foundry-models-mai-thinking)
- [Foundry Models lifecycle and support policy](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirements)
