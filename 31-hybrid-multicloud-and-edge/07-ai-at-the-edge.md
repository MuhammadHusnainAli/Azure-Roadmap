# AI at the Edge

## Why it matters

Not every AI workload can or should call a cloud endpoint. A factory quality camera needs millisecond decisions even when the WAN is down; a hospital's documents may not leave the site; a defence installation is disconnected by design; a retail chain can't stream video from thousands of stores to the cloud economically. Edge AI runs models where the data is: on devices with **Foundry Local**, on on-premises Kubernetes with **Agents and Tools with Foundry Local** (local agentic RAG over on-premises documents), and with Arc-enabled AI services such as **Video Indexer enabled by Arc** and **Azure Machine Learning on Arc**. The cloud still trains, evaluates, governs and updates the models. Architects must decide where each part of an AI system runs and how to operate it across sites.

## Key concepts

### Edge AI options

| Option | Where it runs | What it does | Status |
|---|---|---|---|
| **Foundry Local** | User devices (Windows, macOS, Linux) | On-device inference with a ~20 MB runtime, curated model catalogue (GPT OSS, Qwen, DeepSeek, Mistral, Phi, audio models), automatic GPU/NPU/CPU acceleration, OpenAI-compatible API including Responses format, optional local server; no Azure subscription needed ([Topic 14 file 08](../14-microsoft-foundry-platform/08-foundry-local-and-edge-inference.md)) | Available |
| **Foundry Local on Azure Local** | Arc-enabled Kubernetes on Azure Local | Enterprise-scale local model endpoints with Kubernetes-native operations | Check current status |
| **Agents and Tools with Foundry Local – Agentic Retrieval** (formerly Edge RAG) | Arc-enabled Kubernetes on Azure Local, including disconnected operations | Local agentic RAG: agents with knowledge bases and a built-in MCP server; local document ingestion, embedding (BGE-M3, CLIP on GPUs) and retrieval with Docling parsing; built-in chat UI with citations; image retrieval; BYOM (OpenAI-compatible endpoint) | **Preview** |
| **Azure AI Video Indexer enabled by Arc** | Arc-enabled Kubernetes | Video and audio analysis on-premises | See docs |
| **Azure Machine Learning on Arc** | Arc-enabled Kubernetes | Train and deploy models on your infrastructure (Topic 21) | Available |
| **Containers for Foundry Tools** | Docker/Kubernetes, including disconnected containers | Speech, Language, Translator, Document Intelligence and more running locally ([Topic 20](../20-foundry-tools/README.md)) | Varies by service |
| **Azure IoT Operations** | Arc-enabled Kubernetes | Industrial data plane feeding edge and cloud AI (Topic 32) | Available |

### Edge vs cloud split

| Function | Typical location |
|---|---|
| Real-time inference, data reduction | Edge |
| Sensitive document retrieval and answers | Edge (Agentic Retrieval) when data can't leave |
| Model training, fine-tuning, evaluation at scale | Cloud (Foundry, Azure ML) |
| Fleet management, policy, monitoring | Cloud via Arc |
| Large-model reasoning when allowed | Cloud with local fallback |

### Hybrid routing

Applications can route requests: local small model for routine/offline cases, cloud model for complex queries when connected and permitted, with consistent OpenAI-compatible APIs making the switch simple.

## How it works on Azure

On-device inference with the Foundry Local SDK (Python, following the getting-started sample; `pip install foundry-local-sdk`):

```python
from foundry_local_sdk import (ChatSession, Configuration, FoundryLocalManager, MessageItem,
                               Request, RequestOptions, SearchOptions, TextItem, TextItemType)

FoundryLocalManager.initialize(Configuration(app_name="maintenance-assistant"))
manager = FoundryLocalManager.instance
missing = [ep.name for ep in manager.discover_eps() if not ep.is_registered]
if missing:
    manager.download_and_register_eps(missing)      # GPU/NPU execution providers

model = manager.catalog.get_model("qwen2.5-0.5b")
if not model.is_cached:
    model.download(lambda p: None)
model.load()
try:
    with ChatSession(model) as session:
        session.set_options(RequestOptions(search=SearchOptions(temperature=0.0, max_output_tokens=256)))
        session.set_streaming(True)
        with Request().add_item(MessageItem.user("Summarise: pump P-12 vibration above threshold since 06:00.")) as req:
            with session.process_streaming_request(req) as stream:
                for item in stream:
                    if isinstance(item, TextItem) and item.type == TextItemType.DEFAULT:
                        print(item.text, end="")
finally:
    model.unload()
    manager.close()
```

Foundry Local also exposes an optional OpenAI-compatible local server, so existing OpenAI SDK code can point to it with minimal changes.

Local agentic RAG on Azure Local (outline):

```text
Azure Local (GPU nodes) ─ AKS enabled by Arc ─ Agentic Retrieval extension (Preview)
  ├─ Knowledge layer: ingest SharePoint export / file shares → Docling parse → BGE-M3/CLIP embeddings → local index
  ├─ Agentic layer: agents + knowledge bases + MCP server (local tools, optional external MCP servers)
  ├─ Model: Foundry Local on Azure Local endpoint (recommended) or BYOM OpenAI-compatible endpoint
  └─ Chat UI with citations for users on the site network
Managed from Azure via Arc when connected; disconnected deployment via an imported expansion pack.
```

Fleet operations: GitOps for model and app versions per site; Azure Monitor for edge metrics when connected; evaluation of new model versions in the cloud before rolling to sites in rings.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Device vs site vs cloud** | Device for personal/offline apps; site (Azure Local) for shared, sensitive or high-volume local workloads; cloud for heavy reasoning and training. |
| **Model size** | Small language models fit edge hardware; evaluate quality vs cloud models for each task. |
| **Preview features** | Agentic Retrieval is Preview; pilot, validate and plan for change. |
| **Disconnected** | Plan model updates via expansion packs/offline images; local evaluation and logging. |
| **Hardware** | GPUs for embeddings and LLMs at sites; NPUs on devices; capacity for growth. |
| **Governance** | Same responsible AI controls at the edge: content filtering where available, logging, human oversight, model provenance. |

## Hands-on

1. Install Foundry Local; run a model from the CLI and call it with the OpenAI SDK.
2. Build a hybrid app that uses Foundry Local offline and a Foundry cloud deployment when online.
3. Design an Agentic Retrieval deployment for a plant: GPU nodes, data sources, model endpoint, users.
4. Plan a ring-based rollout of a new model version to 50 sites with GitOps.
5. Compare answer quality of a local SLM and a cloud model on 30 domain questions.

## Security, cost and reliability notes

- **Model and data security** at sites: disk encryption, access control, secure boot, physical security.
- **Safety controls**: cloud guardrails don't apply automatically to local models; add filtering and evaluation.
- **Cost**: no per-token cost locally, but hardware, power and operations; compare TCO with cloud inference.
- **Reliability**: local inference continues during outages; plan fallback when local hardware fails.

## Common mistakes

| Mistake | Fix |
|---|---|
| Sending all raw data to the cloud | Process at the edge |
| Assuming cloud safety controls apply locally | Add local guardrails and evaluation |
| Building production on Preview without a plan | Pilot and track changes |
| Manual model updates per site | GitOps and rings |
| Choosing models without local evaluation | Evaluate on target hardware |

## Check yourself

1. What does Foundry Local provide?
2. What is Agentic Retrieval in Agents and Tools with Foundry Local?
3. Which models does Agentic Retrieval run locally for embeddings?
4. How should work be split between edge and cloud?
5. How do you update models across many sites?
6. Why do edge deployments need their own safety controls?

<details>
<summary>Answers</summary>

1. On-device inference with a small runtime, curated optimised models, automatic hardware acceleration and an OpenAI-compatible API, working offline without an Azure subscription.
2. A Preview Arc-enabled Kubernetes extension providing local agentic RAG on Azure Local: agents, knowledge sources, MCP server and chat UI, including disconnected operation.
3. BGE-M3 for text embeddings and CLIP ViT-L/14 for image embeddings (on GPUs), with Docling parsing on CPU.
4. Edge for real-time inference, data reduction and sensitive retrieval; cloud for training, evaluation, heavy reasoning and fleet management.
5. GitOps with ring-based rollout after cloud evaluation; offline packages for disconnected sites.
6. Cloud guardrails and monitoring don't automatically apply to locally hosted models.

</details>

## References

- [What is Foundry Local?](https://learn.microsoft.com/azure/foundry-local/what-is-foundry-local)
- [Get started with Foundry Local](https://learn.microsoft.com/azure/foundry-local/get-started)
- [Agents and Tools with Foundry Local (Agentic Retrieval)](https://learn.microsoft.com/azure/azure-arc/agents-tools-foundry-local/overview)
- [Video Indexer enabled by Arc](https://learn.microsoft.com/azure/azure-video-indexer/arc/azure-video-indexer-enabled-by-arc-overview)
