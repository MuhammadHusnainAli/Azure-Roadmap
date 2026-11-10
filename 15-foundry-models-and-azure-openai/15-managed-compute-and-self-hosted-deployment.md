# Managed Compute and Self-Hosted Deployment

## Why it matters

Not every model is offered pay-per-token. Open-source models from Hugging Face, fine-tuned open weights, specialised embedding, rerank, vision or speech models, and models you must run in an isolated network often need **dedicated GPU capacity**. Foundry's **managed compute** hosts them without you running Kubernetes or building containers. You can also **self-host** on AKS, Container Apps or VMs for full control. Architects have to decide when that's worth it, because dedicated GPUs cost money whether used or not.

## Key concepts

### Hosting options compared

| Option | What you manage | Billing | Best for |
|---|---|---|---|
| **Serverless (standard / provisioned)** | Nothing below the API | Per token or PTU | Models sold by Azure and partner models: the default |
| **Foundry managed compute** (Preview) | Model choice, deployment template, instance count | **Hourly per accelerator** | Open-source models on dedicated GPUs with Foundry endpoint, auth, networking and observability |
| **Azure Machine Learning managed online endpoints** (GA) | Model package, environment, VM SKU, scaling | Per VM hour | Custom models, MLOps pipelines, classic hub-based managed compute (topic 21) |
| **AKS with GPUs** (vLLM, KAITO, NVIDIA NIM, Triton) | Cluster, node pools, runtime, scaling, patching | VM hours (+ cluster costs) | Full control, multi-model platforms, existing Kubernetes skills (topic 07 file 10) |
| **Container Apps serverless GPUs** | Container image, scaling rules | Per second of GPU use (with scale to zero) | Bursty inference of small and medium open models (topic 07 file 05) |
| **GPU VMs** | Everything | VM hours | Specialised research or legacy setups |
| **Foundry Local** | App packaging | No cloud cost | On-device inference (topic 14 file 08) |

### Foundry managed compute (Preview, checked 2026-11)

| Concept | Detail |
|---|---|
| **Model instance** | Unit of deployment and scaling; you set the number of instances (`capacity`), and Foundry chooses the GPU topology per instance |
| **Deployment template** | Versioned recipe per model: runtime, accelerator family and count per instance, context length, quantisation, parsers (for example `qwen--qwen3-32b--128k-nvidia-2xh100`) |
| **Accelerator families** | NVIDIA **A100 80 GB**, NVIDIA **H100 80 GB**, AMD **MI300X 192 GB** |
| **Runtimes (Microsoft-built, scanned, patched)** | vLLM (default for most LLMs), SGLang (structured output), TensorRT-LLM, NVIDIA NIM, Text Embeddings Inference (embeddings and rerankers), llama.cpp, hf-serve (vision, audio, other pipelines) |
| **Catalogue** | **Hugging Face Collection** (`azure-huggingface` registry): curated weekly, SafeTensors only, no `trust_remote_code`, licences reviewed |
| **Weights** | Pre-staged in Microsoft-managed storage, so **no outbound access to Hugging Face is needed** (works in fully private networks) |
| **Endpoints** | Unified Foundry endpoint: `https://<account>.services.ai.azure.com/managed-deployments/<deployment>/` for all, and `/openai/v1/` for OpenAI-compatible runtimes (deployment name as `model`) |
| **Auth** | Entra ID (scope `https://ai.azure.com/.default`, **Foundry User** role) or account key |
| **Deployment scope** | Global (`GlobalManagedCompute` SKU); data zone and more regions rolling out |
| **Billing** | Accelerators per instance × instances × hours × hourly rate (varies by family) |
| **Quota** | Per **accelerator family per region**, separate from Azure VM quota |
| **RBAC** | Create, update and delete: Cognitive Services Contributor, Foundry Owner or Foundry Account Owner; read: Foundry User or Project Manager |
| **LoRA adapters** | Serve fine-tuned LoRA adapters on compatible base deployments (Preview) |
| **Limitations** | Built-in Content Safety filters **aren't** in the managed compute data path during preview, so call Content Safety yourself if you need it; preview SLA terms |

### When to self-host (or use managed compute)

| Signal | Lean towards dedicated hosting |
|---|---|
| The model isn't available serverless | Yes |
| You need custom or fine-tuned open weights (full fine-tune or LoRA) | Yes |
| Strict isolation: private network, no shared infrastructure | Yes (managed compute in a private network, or AKS) |
| Very high, steady utilisation where per-hour GPU beats per-token pricing | Possibly. Do the maths |
| Specialised serving needs (custom runtimes, speculative decoding, multi-LoRA) | AKS or managed compute templates |

| Signal | Stay serverless |
|---|---|
| Variable or low traffic | Idle GPUs are expensive |
| You need frontier closed models | Only available serverless |
| Small team without GPU operations skills | Serverless removes that burden |
| You need built-in content filters and SLAs | Serverless GA offerings |

### Capacity planning for GPUs

| Factor | Consideration |
|---|---|
| Model memory | Parameters × bytes per parameter (FP16 = 2 bytes; INT8 = 1; INT4 = 0.5) + **KV cache** (grows with context length and concurrency) |
| Throughput | Tokens per second per instance under your prompt and response lengths and concurrency; measure with load tests |
| Context length | Longer contexts need more KV-cache memory, sometimes more GPUs per instance (see templates) |
| Quota and availability | GPU quota is scarce in some regions; request early |
| Scaling | Instances take minutes to start (weights load); keep a minimum warm instance for latency-sensitive use |

Rule of thumb: a 70B-parameter model in FP16 needs about 140 GB for weights alone, before KV cache, so it spans multiple 80 GB GPUs or uses quantisation.

## How it works on Azure

### Deploy and call a managed compute model (sketch)

```bash
# Portal: Model catalogue → Hugging Face collection → choose model → pick a deployment template → Deploy (managed compute)
# CLI/ARM: create an accounts/deployments resource with SKU name GlobalManagedCompute and the chosen template
# (see "Deploy open-source models with managed compute" for the exact properties during preview)
```

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://ai.azure.com/.default")
client = OpenAI(base_url="https://<account>.services.ai.azure.com/openai/v1/", api_key=tp)

r = client.chat.completions.create(model="<managed-compute-deployment-name>",
                                   messages=[{"role": "user", "content": "Summarise Kubernetes in two sentences."}])
print(r.choices[0].message.content)
```

The same client code works for serverless and managed compute deployments. Only the deployment name changes (for OpenAI-compatible runtimes).

### Self-hosting on AKS (outline)

```text
AKS cluster ─ GPU node pool (e.g. NC/ND H100 or A100 VMs)
   ├─ KAITO workspace (Kubernetes AI Toolchain Operator) → provisions GPU nodes and a vLLM inference service
   ├─ or NVIDIA NIM / vLLM Helm charts
   ├─ Ingress/internal load balancer → APIM AI gateway (auth, token limits, logging)
   └─ Azure Monitor + Prometheus (GPU utilisation, tokens/s, latency)
Content safety: call Azure AI Content Safety from the gateway or app
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Managed compute vs AKS | Managed compute for "just host this open model" with Foundry integration and Microsoft-patched runtimes; AKS for multi-model platforms, custom runtimes or existing Kubernetes operations |
| Managed compute (preview) vs Azure ML online endpoints (GA) | Azure ML endpoints when you need GA SLAs, custom inference code and MLOps pipelines today; managed compute for the simplest open-model hosting on the unified Foundry endpoint |
| GPU family | H100 for latency and throughput, A100 for cost-effective capacity, MI300X for very large memory (big models and long contexts in fewer GPUs) |
| Quantisation | Use templates with quantisation to fit models in fewer GPUs when quality allows; evaluate the quality impact |
| Safety | Add Content Safety (Prompt Shields, harm categories) in front of self-hosted models |
| Cost control | Scale to zero or minimal instances off-hours where latency allows; consolidate models with multi-LoRA; compare with serverless pricing quarterly |

## Hands-on

1. Browse the Hugging Face collection in the Foundry catalogue and compare two deployment templates for the same model (accelerator, context length).
2. Deploy a small open model to managed compute with one instance, call it with the OpenAI SDK via `/openai/v1`, then delete the deployment.
3. Estimate GPU memory for a 32B model at FP16 and INT4 with a 32K context, and choose a template.
4. Build a cost comparison: serverless per-token vs managed compute per hour at 10%, 50% and 90% utilisation.
5. Optional: deploy a model with KAITO on AKS and put API Management in front of it.

## Security, cost and reliability notes

- **Security:** use private networking and Entra ID; managed compute weights are pre-staged, so no internet egress to Hugging Face is needed.
- **Security:** built-in content filters aren't applied on managed compute (preview); add Content Safety and evaluations yourself.
- **Licensing:** review each open model's licence (surfaced on the model card) for commercial use and redistribution.
- **Cost:** you pay per accelerator-hour while deployed, idle or not. Delete or scale down test deployments promptly.
- **Reliability:** keep at least two instances for production availability; plan for preview SLA limitations, or use Azure ML endpoints or AKS for GA SLAs.

## Common mistakes

- Self-hosting a model that's available serverless at lower total cost.
- Forgetting KV-cache memory when sizing GPUs, then hitting out-of-memory at high concurrency.
- Assuming built-in content filters protect managed compute deployments.
- Leaving GPU deployments running over weekends for experiments.
- Ignoring model licences for open weights.

## Check yourself

1. Name four ways to host models on Azure besides serverless deployments.
2. What is a deployment template in Foundry managed compute?
3. Which accelerator families does managed compute support?
4. Which endpoint routes can you use to call a managed compute deployment?
5. How is managed compute billed and how is its quota scoped?
6. What safety gap exists in managed compute during preview?
7. When is self-hosting justified?

<details>
<summary>Answers</summary>

1. Any four of: Foundry managed compute, Azure ML managed online endpoints, AKS with GPUs (vLLM, KAITO, NIM, Triton), Container Apps serverless GPUs, GPU VMs, Foundry Local on devices.
2. A versioned recipe that pins the runtime, accelerator family and count per instance, context length, quantisation and runtime tuning for a specific model.
3. NVIDIA A100 80 GB, NVIDIA H100 80 GB and AMD MI300X 192 GB.
4. `https://<account>.services.ai.azure.com/managed-deployments/<deployment>/` for every deployment, and `/openai/v1/` for OpenAI-compatible runtimes.
5. Hourly per accelerator (accelerators per instance × instances × hours × rate); quota is per accelerator family per region, separate from VM quota.
6. Built-in Azure AI Content Safety filters aren't in the data path; call Content Safety yourself.
7. When the model isn't available serverless, you need custom or fine-tuned open weights, strict isolation, specialised serving, or very high steady utilisation that makes dedicated GPUs cheaper.

</details>

## References

- [Managed compute in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/managed-compute-overview)
- [Deploy open-source models with managed compute](https://learn.microsoft.com/azure/foundry/how-to/deploy-models-managed)
- [Role-based access control: managed compute operations](https://learn.microsoft.com/azure/foundry/concepts/rbac-foundry#managed-compute-control-plane-operations)
- [Azure ML managed online endpoints](https://learn.microsoft.com/azure/machine-learning/concept-endpoints-online)
- [Kubernetes AI Toolchain Operator (KAITO) on AKS](https://learn.microsoft.com/azure/aks/ai-toolchain-operator)
- [Serverless GPUs in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/gpu-serverless-overview)
