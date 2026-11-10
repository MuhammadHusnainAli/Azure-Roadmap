# 15 · Foundry Models and Azure OpenAI

> Tracks: E ● A ● C ○ · Part D · AI on Azure

Models are the engine of every AI solution, and Foundry offers a very large catalogue: Azure OpenAI (GPT-6, GPT-5.x, o-series, gpt-oss, image, video, audio and realtime models), Anthropic Claude, Microsoft MAI and Phi, Meta Llama, Mistral, xAI Grok, DeepSeek, Cohere, Moonshot Kimi, Black Forest Labs, Fireworks-hosted and Hugging Face open models. This topic covers how to choose, deploy, call and operate them: catalogue categories and terms, deployment types and data residency, provisioned throughput, the model router, the Chat Completions and Responses APIs, function calling and structured outputs, embeddings, vision, image and video generation, audio and realtime, batch and caching, rate limits and resilience, the model lifecycle, and managed compute for open models.

## What you will be able to do

- Distinguish models sold by Azure from partner and community models, and explain terms, billing and support.
- Navigate the Azure OpenAI model families and use the v1 API with keyless authentication.
- Call non-OpenAI models (Claude via the Anthropic SDK; others via Chat Completions).
- Choose a deployment type (Global, Data Zone, Regional × Standard, Provisioned, Batch) and explain data residency precisely.
- Size provisioned throughput, use spillover, and decide when PTU beats pay-as-you-go.
- Select models with benchmarks and evaluations, and configure the model router.
- Build with Chat Completions and the Responses API: streaming, stateful turns, background mode, MCP tools.
- Implement function calling and strict structured outputs safely.
- Choose and operate embedding models, vision, image and video generation, and realtime voice.
- Cut cost with batch, prompt caching, priority and flex processing, and token budgeting.
- Design for 429s with retries, backoff, multi-region back ends and gateways.
- Plan for model deprecation, retirement and upgrades.
- Host open models on managed compute or self-host when justified.

## Prerequisites

[Topic 13](../13-ai-and-generative-ai-foundations/README.md) (LLMs, tokens, embeddings) and [Topic 14](../14-microsoft-foundry-platform/README.md) (Foundry resources, roles, SDKs). [Topic 11](../11-integration-messaging-and-apis/README.md) file 07 (AI gateway) helps for files 05 and 13.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Model Catalogue](01-model-catalogue.md) | 40 min | Portal, CLI |
| 02 | [Azure OpenAI in Foundry Models](02-azure-openai-in-foundry-models.md) | 50 min | Python |
| 03 | [Other Model Families](03-other-model-families.md) | 45 min | Python (Anthropic, OpenAI SDKs) |
| 04 | [Deployment Types](04-deployment-types.md) | 45 min | CLI, Azure Policy |
| 05 | [Provisioned Throughput](05-provisioned-throughput.md) | 50 min | Calculator, load test |
| 06 | [Model Router and Model Selection](06-model-router-and-model-selection.md) | 45 min | Evaluations, Python |
| 07 | [Chat Completions and Responses API](07-chat-completions-and-responses-api.md) | 60 min | Python |
| 08 | [Function Calling and Structured Outputs](08-function-calling-and-structured-outputs.md) | 55 min | Python |
| 09 | [Embedding Models](09-embedding-models.md) | 40 min | Python, AI Search |
| 10 | [Vision, Image and Video Generation](10-vision-image-and-video-generation.md) | 40 min | Python |
| 11 | [Audio and Real-Time APIs](11-audio-and-real-time-apis.md) | 50 min | Python, WebRTC |
| 12 | [Batch, Prompt Caching and Cost Optimisation](12-batch-prompt-caching-and-cost-optimisation.md) | 50 min | Python |
| 13 | [Rate Limits, Quotas and Resilience](13-rate-limits-quotas-and-resilience.md) | 45 min | Python, APIM |
| 14 | [Model Lifecycle](14-model-lifecycle.md) | 35 min | REST, Bicep |
| 15 | [Managed Compute and Self-Hosted Deployment](15-managed-compute-and-self-hosted-deployment.md) | 45 min | Portal, Python |

**AI engineers:** all of it, with 02, 07, 08, 09, 12 and 13 in depth. **Architects:** all of it, with 01, 04, 05, 06, 13, 14 and 15 in depth. **Consultants:** 01, 03, 04, 05, 12 and 14.

## Completion checklist

- [ ] I can explain the difference between models sold by Azure and models from partners, including Claude's hosting variants.
- [ ] I have called a model through the v1 endpoint with Entra ID, and Claude through the Anthropic SDK.
- [ ] I can state exactly where inference and data at rest are processed for each deployment type.
- [ ] I can size a PTU deployment and configure spillover.
- [ ] I have compared models with my own evaluation set and tried the model router modes.
- [ ] I have built a stateful Responses conversation, a streaming UI and a background task.
- [ ] I have implemented a function-calling loop with validation and a strict structured output with Pydantic.
- [ ] I have chosen embedding dimensions based on measured recall.
- [ ] I have generated an image, used vision input and run a realtime voice session with ephemeral tokens.
- [ ] I have run a batch job and measured prompt cache hits.
- [ ] I have handled 429s with backoff and set up multi-region failover through a gateway.
- [ ] I know the lifecycle stages, `versionUpgradeOption` values and retirement dates of my deployments.
- [ ] I can decide when to use managed compute or self-hosting instead of serverless.

## Maps to

- **AI-103**: deploy and consume models, choose deployment types, implement structured outputs, tools, embeddings, vision, audio and cost controls.
- **AI-200**: integrate Azure OpenAI and Foundry Models into applications.
- **AZ-305**: design AI model hosting, residency, capacity and resilience.
- **AI-901**: model families and deployment concepts (foundational).

## Next

16 · Prompt Engineering and Model Customisation (see [Roadmap.md](../Roadmap.md))
