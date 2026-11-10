# Small Language Models and Open Models

> Topic: 13 · AI and Generative AI Foundations · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

Not every problem needs a frontier model in the cloud. **Small language models (SLMs)** such as Microsoft's **Phi** family can classify, extract, summarise and route at a fraction of the cost and latency, and they can run **on devices** (laptops, phones, factory PCs) where data must stay local or connectivity is poor. **Open-weight models** (Llama, Mistral, DeepSeek, gpt-oss, Phi and others) give you the weights, so you can fine-tune, self-host and control them. Architects and consultants have to weigh these options against hosted frontier models on cost, quality, privacy, control and operations.

## Key concepts

### Definitions

| Term | Meaning |
|---|---|
| **Small language model (SLM)** | Informally, a model of a few hundred million to roughly 15 billion parameters, small enough to run on a single GPU, a laptop or sometimes a phone |
| **Frontier model** | The most capable large models (for example the newest GPT, Claude, Grok and Gemini generations), usually only available as a hosted service |
| **Open-weight model** | Weights published for download under a licence; the training data and code may not be open |
| **Open-source model** | Weights *and* (ideally) training code and data under an OSI-style licence. Many "open" models are open-weight only |
| **Proprietary / closed model** | Weights not released; available only via API |

**Licences differ.** MIT (Phi), Apache 2.0 (several Mistral models, gpt-oss), and custom licences with use restrictions (Llama community licence). Always read the licence and acceptable use policy before commercial use.

### The Phi family

Microsoft's **Phi** models are SLMs trained heavily on high-quality and synthetic "textbook-like" data, showing that careful data can substitute for some scale. As of late 2026 the Phi-4 generation includes (check the catalogue for the current list):

| Model (examples) | Notes |
|---|---|
| **Phi-4** (about 14B) | General reasoning, strong for its size |
| **Phi-4-mini** (about 3.8B) | Small, fast, function calling; good for on-device and high-volume tasks |
| **Phi-4-multimodal** | Text, image and audio input in one small model |
| **Phi-4-reasoning / Phi-4-mini-reasoning** | Reasoning-tuned variants for maths and logic |
| Newer reasoning-vision variants | Released during 2026; check the catalogue |

Phi weights are published on Hugging Face under the MIT licence and are available in the Foundry catalogue (serverless and/or managed compute) and through **Foundry Local** for on-device use. Older generations (Phi-3) are being retired from Foundry hosting, so check the retirement schedule.

### Other open-weight families on Foundry (examples, checked 2026-11)

| Family | Provider | Notes |
|---|---|---|
| **gpt-oss** (`gpt-oss-120b`, `gpt-oss-20b`) | OpenAI | Open-weight reasoning models (Apache 2.0) |
| **Llama** (Llama 3.3, Llama 4) | Meta | Llama community licence |
| **Mistral** | Mistral AI | Mix of open-weight and commercial models |
| **DeepSeek** (V3.x, V4 series, R1 lineage) | DeepSeek | Reasoning and MoE models; review data, safety and policy considerations |
| **Qwen, Kimi** and others | Various | Availability varies by catalogue section |

Some are "sold directly by Azure" (Microsoft provides support and SLAs); others come "from partners and community" (topic 15).

### Where SLMs win and lose

| SLMs are a good fit | Prefer larger models |
|---|---|
| Classification, routing, intent detection, extraction with clear schemas | Open-ended reasoning across many domains |
| High-volume, cost-sensitive workloads | Complex multi-step agentic planning with many tools |
| Low-latency or offline scenarios | Long-context synthesis of many documents |
| On-device privacy (data never leaves the device) | Nuanced writing, broad world knowledge |
| Fine-tuning for a narrow domain | Multilingual breadth (varies by model) |

A common pattern is a **cascade**: an SLM handles easy or classification requests and escalates hard ones to a frontier model. The Foundry **model router** automates a version of this for hosted models (topic 15).

### Making models smaller and faster

| Technique | What it does |
|---|---|
| **Quantisation** | Store weights in lower precision (FP16 → INT8/INT4), cutting memory and speeding inference with small quality loss |
| **Distillation** | Train a small "student" model to imitate a large "teacher" (Foundry supports distillation workflows; topic 16) |
| **Pruning** | Remove less important weights or layers |
| **Optimised runtimes** | ONNX Runtime, vLLM, TensorRT-LLM, llama.cpp; hardware-specific acceleration on CPU, GPU and NPU |

## How it works on Azure

### Deployment options for open and small models

| Option | Where it runs | Billing | When |
|---|---|---|---|
| **Serverless / standard deployment** (models sold by Azure or partners) | Microsoft-managed | Per token | Simplest; no GPU management |
| **Managed compute** (Foundry / Azure ML online endpoints) | Dedicated VMs (GPU) in your subscription's quota | Per VM hour | Models not offered serverless, custom or fine-tuned weights, isolation |
| **AKS / Container Apps with GPUs** (vLLM, KAITO) | Your clusters | Infrastructure cost | Full control, existing Kubernetes platform (topic 07) |
| **Foundry Local** | User's device (Windows, macOS) | No cloud cost | Offline, private, low-latency on-device inference |
| **Windows AI APIs / Windows ML** | Copilot+ PCs and Windows devices (NPU, GPU, CPU) | No cloud cost | Windows apps using built-in or bundled models |

### Foundry Local

Foundry Local (GA since April 2026) runs models on a user's device with hardware acceleration (CPU, GPU, NPU through ONNX Runtime), exposing an **OpenAI-compatible** local endpoint and SDKs, so the same client code can target cloud or device.

```bash
# Windows (winget) or macOS (brew); check the install docs for the current package name
winget install Microsoft.FoundryLocal
foundry model list                 # models available for your hardware
foundry model run phi-4-mini       # downloads (if needed) and starts an interactive session
foundry service status             # shows the local endpoint URL
```

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:<port>/v1", api_key="not-needed-locally")
r = client.chat.completions.create(model="phi-4-mini", messages=[{"role": "user", "content": "Classify: 'printer jammed again' → hardware|software|other"}])
print(r.choices[0].message.content)
```

Model aliases, ports and package names change between releases. Use `foundry model list` and `foundry service status`. Some CLI features may still be marked preview even though the product is GA. Topic 14 file 08 covers Foundry Local in depth.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| SLM vs frontier model | Run both on your evaluation set; choose the cheapest model that meets the quality bar. Re-check when new models release |
| Hosted vs self-hosted open model | Hosted (serverless) unless you need custom weights, specific hardware, strict isolation or very high steady utilisation that makes dedicated GPUs cheaper |
| On-device vs cloud | On-device for offline, privacy-sensitive, latency-critical or very high-volume edge workloads; cloud for heavy reasoning and shared knowledge. Hybrid patterns route between them |
| Licence | Confirm commercial terms, attribution and acceptable use; involve legal for custom licences |
| Fine-tune an SLM vs prompt a large model | Fine-tune an SLM for narrow, stable, high-volume tasks with good training data; prompt or RAG a large model for broad or changing tasks |

## Hands-on

1. Deploy `Phi-4-mini` (or another SLM) and a frontier model in Foundry. Run 50 classification examples through both; compare accuracy, latency and cost per 1,000 requests.
2. Install Foundry Local, run `phi-4-mini` on your laptop, and call it with the OpenAI SDK. Disconnect from the network and confirm it still works.
3. Read the licences for Phi, Llama and gpt-oss models and summarise the commercial-use differences in a table.
4. Design a cascade: SLM for intent classification → frontier model only for complex intents. Estimate the cost saving for 1 million requests a month.

## Security, cost and reliability notes

- **Security:** on-device models ship weights and prompts to endpoints you don't control, so consider model tampering and local data protection. Don't put secrets in on-device prompts.
- **Security:** open-weight models may have weaker safety tuning than hosted models with Azure AI Content Safety. Add content filtering and evaluation yourself when self-hosting.
- **Cost:** SLMs cut per-request cost dramatically, but dedicated GPU hosting is billed whether or not it is used, so check utilisation before self-hosting.
- **Reliability:** self-hosted models need capacity planning, autoscaling, updates and monitoring that serverless handles for you.

## Common mistakes

- Treating "open-weight" as "open source with no restrictions".
- Choosing a frontier model for simple classification at scale.
- Self-hosting a model on GPUs that sit idle most of the day.
- Assuming an SLM matches frontier quality without evaluating it on your own data.
- Forgetting that safety filtering isn't automatic when you self-host.

## Check yourself

1. What is the difference between open-weight and open-source?
2. Name three tasks where an SLM is often sufficient.
3. What is Phi's main idea for achieving strong performance at small size?
4. Name four ways to deploy an open model on Azure or on devices.
5. What do quantisation and distillation do?
6. When is self-hosting cheaper than serverless?

<details>
<summary>Answers</summary>

1. Open-weight models publish weights (under some licence); open-source implies weights plus code and ideally data under an open licence. Many models are open-weight only.
2. Classification and routing, structured extraction, short summarisation (also on-device assistants, simple tool calling).
3. Training on carefully curated high-quality and synthetic "textbook-like" data, so data quality substitutes for some scale.
4. Serverless/standard deployment in Foundry; managed compute endpoints; AKS or Container Apps with GPUs (vLLM, KAITO); Foundry Local or Windows ML on devices.
5. Quantisation stores weights at lower precision to reduce memory and speed up inference; distillation trains a smaller model to imitate a larger one.
6. When traffic is high and steady enough to keep dedicated GPUs well utilised, or when you need custom weights or hardware that serverless doesn't offer.

</details>

## References

- [Phi open models (Microsoft)](https://azure.microsoft.com/products/phi)
- [Foundry Models from partners and community](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-from-partners)
- [What is Foundry Local?](https://learn.microsoft.com/azure/foundry-local/what-is-foundry-local)
- [Foundry Local CLI reference](https://learn.microsoft.com/azure/foundry-local/reference/reference-cli)
- [Foundry Local is generally available (Foundry blog, April 2026)](https://devblogs.microsoft.com/foundry/foundry-local-ga/)
- [Windows AI and Windows ML](https://learn.microsoft.com/windows/ai/)
- [Deploy models to managed compute](https://learn.microsoft.com/azure/foundry/how-to/deploy-models-managed)
