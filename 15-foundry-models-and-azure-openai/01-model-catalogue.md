# Model Catalogue

## Why it matters

The Foundry model catalogue lists thousands of models from OpenAI, Anthropic, Microsoft, Meta, Mistral, xAI, DeepSeek, Cohere, Moonshot, Black Forest Labs, Fireworks AI, NVIDIA, Hugging Face and others. They differ in more than quality and price. Who sells and supports a model, which licence and terms apply, how it's billed, which deployment types and regions it supports, and whether data stays inside Azure all vary. These differences matter for procurement, compliance and support escalation, so architects and consultants have to understand them before recommending a model.

## Key concepts

### Two categories of Foundry Models

| | **Foundry Models sold by Azure** ("sold directly by Azure") | **Foundry Models from partners and community** |
|---|---|---|
| Who sells | Microsoft | The model provider, through **Azure Marketplace** (or community/open models you host) |
| Terms | Microsoft Product Terms; covered by Microsoft's enterprise agreements and data protection commitments | Partner models not sold by Azure are **Non-Microsoft Products** under the Product Terms; the provider sets licence terms and price |
| Billing | Azure meters on your Azure bill | Marketplace offer (SaaS subscription) billed through Azure; open models hosted on managed compute billed per accelerator-hour |
| Support | Microsoft support | Provider-dependent; Microsoft supports the platform |
| Examples (2026-11) | Azure OpenAI models (GPT-6, GPT-5.x, o-series, gpt-oss, embeddings, gpt-image, sora-2, realtime and audio), Microsoft MAI models, model router, Black Forest Labs FLUX, Cohere (command, embed, rerank), DeepSeek, Meta Llama (some), Mistral (some), Moonshot Kimi, xAI Grok | **Anthropic Claude** (Opus, Sonnet, Haiku and newer families), Phi models, Cohere embed v3, Meta Llama 4 Scout, Mistral (Codestral, Ministral, small/medium), NTT tsuzumi, many Hugging Face and open models |
| Purchase requirements | Standard Azure subscription | Marketplace purchase permissions (Owner or Contributor include them), a supported billing country, and no CSP, student or credit-only subscriptions for many offers |

The same provider can appear in both categories (for example Mistral, Meta, Cohere and Microsoft have models in each). Always check which one a specific model is in.

### Deployment options for catalogue models

| Option | What it is | Billing |
|---|---|---|
| **Serverless API / standard deployment** | Microsoft-hosted, pay per token, using deployment types such as Global Standard, Data Zone Standard, Standard, Provisioned, Batch (file 04) | Per token (or PTU) |
| **Managed compute** (Preview) | Open-source models (Hugging Face collection) on dedicated GPUs, with Foundry choosing the topology from a deployment template | Per accelerator-hour (file 15) |
| **Instant access** (Preview) | Call supported models by name with no deployment | Per token |

Some models still require a **hub-based project** (classic) for deployment. Selecting them in the catalogue opens the classic experience.

### Anthropic Claude in Foundry

Claude models are available in Foundry through Azure Marketplace in two variants: **hosted on Azure** and **hosted on Anthropic infrastructure**. Not every model has both, and lifecycle status can differ between them. Points to know (checked 2026-11):

- A paid subscription with a billing account in a country where Anthropic offers the models; CSP, student, free-trial and credit-only subscriptions aren't supported.
- Claude uses the **Messages API** (Anthropic SDK against `https://<resource>.services.ai.azure.com/anthropic`), and some newer Claude models restrict sampling parameters (for example `temperature` and `top_k` aren't supported and `top_p` is fixed).
- Data Zone Standard (US) is available for some Claude models hosted on Azure.
- Partner models such as Anthropic, DeepSeek, Fireworks and Mistral follow a **12-month** GA lifecycle rather than the standard 18 months (file 14).

### Model cards, benchmarks and collections

Each catalogue entry has a **model card**: description, intended use, limitations, licence, context window, modalities, supported deployment types and regions, and often evaluation results and responsible AI notes. The catalogue also offers:

- **Filters**: by provider, task (chat, embeddings, image, audio), deployment option, licence, inference task.
- **Benchmarks and leaderboards**: quality, safety, cost and throughput comparisons (file 06).
- **Collections**: curated sets (for example "Models sold by Azure", provider collections).

## How it works on Azure

### Find and deploy a model

```bash
# What can I deploy in this region? (models sold by Azure and supported partner models)
az cognitiveservices model list -l eastus2 \
  --query "[?kind=='AIServices'].{name:model.name, version:model.version, format:model.format}" -o table

# Deploy a model sold by Azure
az cognitiveservices account deployment create -g rg-foundry -n fdy-contoso-dev \
  --deployment-name gpt-5-mini --model-format OpenAI --model-name gpt-5-mini --model-version 2025-08-07 \
  --sku-name GlobalStandard --sku-capacity 50
```

Partner models from Marketplace are usually deployed from the portal first (to accept the offer terms); afterwards, IaC can manage them. The model `format` value identifies the provider (for example `OpenAI`, `Anthropic`, `Microsoft`, `Meta`, `Mistral AI`, `xAI`, `DeepSeek`, `Cohere`), so check `az cognitiveservices model list` for the exact string.

### Calling different providers from one resource

| Provider family | Typical API |
|---|---|
| Azure OpenAI and most models sold by Azure | OpenAI v1 endpoint (Chat Completions or Responses) |
| Anthropic Claude | Anthropic Messages API (`/anthropic` endpoint) |
| Many partner models | Chat Completions-compatible API on the Foundry endpoint |
| Foundry Agent Service | Supports a subset of models; check "Models supported by Agent Service" |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Sold by Azure vs partner | Prefer models sold by Azure when Microsoft terms, support and data-protection commitments matter (regulated industries, procurement simplicity). Use partner models when they give clear capability or cost advantages and legal has reviewed the terms |
| Hosted on Azure vs provider infrastructure (Claude) | Hosted on Azure when data must stay in Azure regions or data zones; check feature parity and lifecycle |
| Serverless vs managed compute | Serverless for most use; managed compute for models only available that way, custom or fine-tuned open weights, or isolation (file 15) |
| Open vs closed weights | Open weights for portability, fine-tuning and self-hosting; closed models for frontier capability (topic 13 file 06) |
| Breadth vs standardisation | Approve a short list of models per use-case class (frontier reasoning, fast chat, embeddings, vision) and govern it with Azure Policy and the Control Plane |

## Hands-on

1. Browse the catalogue and filter by "Models sold by Azure" and by "from partners". Pick one of each and compare their model cards (licence, deployment types, regions).
2. Deploy one model sold by Azure from the CLI.
3. Deploy a partner model (for example a Claude or Mistral model) from the portal and note the Marketplace offer and terms.
4. List deployments and their `format` values with `az cognitiveservices account deployment list`.
5. Write an Azure Policy (or use the built-in allowed-models policy) that restricts deployments to an approved list.

## Security, cost and reliability notes

- **Security and compliance:** confirm where data is processed for each model (deployment type, hosted on Azure vs provider infrastructure) and the provider's data-use terms.
- **Security:** Content Safety and guardrails apply to many models, but filtering behaviour can differ by model. Check per model.
- **Cost:** Marketplace partner pricing differs from Azure meters and may not count towards Azure commitments in the same way; check with procurement.
- **Reliability:** partner models can have shorter lifecycles (12 months), so plan upgrades.

## Common mistakes

- Assuming every model in the catalogue carries Microsoft's terms and support.
- Choosing a model that isn't available in the required region or deployment type.
- Using a CSP or credit-only subscription and being unable to buy partner models.
- Ignoring licence restrictions on open-weight models.
- Treating leaderboard scores as proof of fitness for your task.

## Check yourself

1. What are the two categories of Foundry Models, and how do terms and billing differ?
2. Which category are Anthropic Claude models in, and what two hosting variants exist?
3. Name three deployment options for catalogue models.
4. Why might some models open in the classic portal?
5. Which subscription types typically can't purchase partner models?
6. What's on a model card?

<details>
<summary>Answers</summary>

1. Models sold by Azure (Microsoft-sold, Azure billing, Microsoft terms and support) and models from partners and community (provider-sold via Azure Marketplace, Non-Microsoft Product terms set by the provider, provider support).
2. Models from partners (Marketplace); hosted on Azure and hosted on Anthropic infrastructure.
3. Serverless API/standard deployments (pay per token or PTU), managed compute (per accelerator-hour), and instant access (preview, no deployment).
4. They still require a hub-based (classic) project for deployment.
5. CSP, student, Visual Studio Enterprise, free-trial and credit-only subscriptions (per the offer requirements).
6. Description, intended uses and limitations, licence, modalities, context window, supported deployment types and regions, benchmarks, and responsible AI notes.

</details>

## References

- [Foundry Models overview](https://learn.microsoft.com/azure/foundry/concepts/foundry-models-overview)
- [Foundry Models sold by Azure](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Foundry Models from partners and community](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-from-partners)
- [Claude models in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/claude-models)
- [Deployment overview for Foundry Models](https://learn.microsoft.com/azure/foundry/concepts/deployments-overview)
- [Instant access to models (preview)](https://learn.microsoft.com/azure/foundry/concepts/instant-models)
