# AKS for AI Workloads

## Why it matters

When an organisation decides to self-host models, fine-tune at scale or run many AI services on a shared platform, AKS is usually where it lands. Kubernetes brings scheduling, isolation and a large ecosystem of AI tooling (vLLM, KubeRay, NVIDIA operators, KAITO). It also brings GPU scarcity, large images, driver management and cost risk. This file covers how to run AI on AKS well, and repeats the question from [topic 06](../06-compute/07-gpu-and-ai-infrastructure.md): should you be self-hosting at all?

## Key concepts

### Typical AI workloads on AKS

| Workload | Shape |
|---|---|
| Open-model inference (LLMs, embeddings, rerankers) | GPU deployments behind an internal service, autoscaled on queue depth or latency |
| Small model and classic ML inference | CPU or small-GPU deployments |
| Agent services and MCP servers | Ordinary CPU deployments calling Foundry or self-hosted models |
| Fine-tuning (LoRA and similar) | GPU jobs, often on Spot with checkpoints |
| Distributed training and batch processing | Ray, PyTorch distributed (Kubeflow Training Operator), Volcano or Kueue for queueing |
| RAG ingestion pipelines | Event-driven CPU workers scaled by KEDA |

### GPU node pools

```bash
az aks nodepool add -g $RG --cluster-name $AKS -n gpua10 \
  --node-vm-size Standard_NC24ads_A100_v4 --node-count 0 \
  --enable-cluster-autoscaler --min-count 0 --max-count 3 \
  --node-taints sku=gpu:NoSchedule --labels workload=llm --zones 1 2 3
```

| Topic | Guidance |
|---|---|
| **Drivers** | AKS installs NVIDIA drivers on GPU node pools by default. Alternatively skip driver install and use the **NVIDIA GPU Operator** to manage drivers, device plugin and DCGM exporter yourself |
| **Device plugin** | Exposes `nvidia.com/gpu` as a schedulable resource; pods request `resources.limits: nvidia.com/gpu: 1` |
| **Taints and tolerations** | Taint GPU pools so only GPU workloads land there |
| **Multi-Instance GPU (MIG)** | Split supported GPUs (A100, H100) into isolated slices for small models (`--gpu-instance-profile`) |
| **Time-slicing** | Share a GPU between pods without isolation, for dev and light loads |
| **Scale to zero** | Minimum 0 with cluster autoscaler or NAP; expect minutes to provision |
| **Quota and capacity** | Same constraints as GPU VMs; capacity reservations for production |
| **Images** | GPU images are large; use ACR artifact streaming and node-local caching |
| **Monitoring** | DCGM exporter → managed Prometheus → Grafana: GPU utilisation, memory, temperature, Xid errors |

### KAITO: the AI toolchain operator

**KAITO** (Kubernetes AI Toolchain Operator) is an open-source CNCF project available as a managed AKS add-on. It turns "deploy this model" into one custom resource:

```yaml
apiVersion: kaito.sh/v1beta1
kind: Workspace
metadata: { name: workspace-phi-4-mini }
resource:
  instanceType: "Standard_NC24ads_A100_v4"
  labelSelector: { matchLabels: { apps: phi-4-mini } }
inference:
  preset: { name: phi-4-mini-instruct }
```

KAITO then:

1. Provisions GPU nodes of the requested size (through node auto-provisioning or its own provisioner).
2. Pulls a pre-built model image or downloads weights.
3. Configures and runs an inference engine (vLLM by default for supported models) with an OpenAI-compatible API.
4. Exposes a Kubernetes Service for the endpoint.

It also supports **fine-tuning** workspaces (LoRA and QLoRA with your data), **RAG engine** workspaces, custom models from Hugging Face, and multi-node inference for large models. Preset models and API versions change frequently; check the KAITO repository for the current list.

### Inference serving stacks

| Stack | Notes |
|---|---|
| **vLLM** | High-throughput LLM serving, continuous batching, paged attention, OpenAI-compatible API |
| **NVIDIA NIM** | Optimised, supported containers for many models (licensing applies) |
| **NVIDIA Triton** | Multi-framework serving, model ensembles |
| **KServe** | Kubernetes-native model serving abstraction, autoscaling and canaries |
| **Ollama** | Simple, good for dev; not optimised for high concurrency |
| **Text Embeddings Inference**, **Infinity** | Embedding and reranker models |

### Ray on AKS

**Ray** is a distributed computing framework for Python widely used for distributed training, hyperparameter tuning, batch inference and serving (Ray Serve). On AKS it runs through **KubeRay** (`RayCluster`, `RayJob`, `RayService` CRDs). Microsoft and Anyscale also offer a managed Ray experience integrated with Azure; check its current availability. Use Ray when Python workloads need to scale across many nodes; do not introduce it for single-node work.

### Batch scheduling and quotas for GPU jobs

Kubernetes' default scheduler is not a batch scheduler. For shared GPU clusters add a queueing layer:

- **Kueue**: job queueing, fair sharing and quotas per team.
- **Volcano**: gang scheduling (all or nothing) for distributed training.

### Gateway in front of models

Put an **AI gateway** between applications and model endpoints, whether self-hosted or Foundry: Azure API Management with its AI gateway policies (token limits, semantic caching, load balancing, metrics) or a Kubernetes gateway with inference extensions (the Gateway API Inference Extension adds model-aware routing). Applications then use one OpenAI-compatible endpoint while you route between self-hosted models and Foundry Models.

## How it works on Azure

### Reference architecture: self-hosted inference on AKS

```text
Apps / agents ──► API Management (AI gateway: auth, token quotas, logging, routing)
                       ├──► Foundry Models (default, frontier)
                       └──► AKS internal ingress (Gateway API)
                              ├─ KAITO workspace: phi-4-mini on A100 (min 2, zones 1·2)
                              ├─ embeddings deployment (T4 MIG slices)
                              └─ reranker deployment (CPU)
                   GPU node pools: tainted, NAP or autoscaler min 0 for batch pools
                   Weights: Blob or ACR OCI artefacts → local NVMe cache
                   Observability: DCGM + vLLM metrics → managed Prometheus → Grafana; traces to App Insights
                   Safety: Azure AI Content Safety called by gateway or app
```

### Choosing a hosting option for open models

| Requirement | Prefer |
|---|---|
| Model in Foundry catalogue as serverless | Foundry Models (no infrastructure) |
| Dedicated capacity for a catalogue model, minimal ops | Foundry managed compute / Azure ML online endpoints |
| Intermittent custom inference | Container Apps serverless GPU |
| Many models, Kubernetes platform team, custom serving | **AKS + KAITO or vLLM** |
| Distributed training and research workloads | AKS with Ray or Kubeflow, or Azure Machine Learning |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| KAITO or hand-built serving | KAITO for supported models and fast start; hand-built vLLM or KServe for full control |
| GPU sharing | MIG for isolated small workloads, whole GPUs for large models, time-slicing for dev only |
| Driver management | AKS-managed drivers by default; GPU Operator when you need specific versions or features |
| Batch vs serving pools | Separate pools: serving on on-demand (reserved) capacity, batch and fine-tuning on Spot |
| Model storage | Weights outside images when large; cache on local NVMe |
| Gateway | Always put a gateway in front; never expose model pods directly |

## Hands-on

GPU quota is needed for steps 2 and 3. Step 1 works without it.

```bash
RG=rg-aks-ai; AKS=aks-ai; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Cluster with OIDC, workload identity and the KAITO add-on
az aks create -g $RG -n $AKS --node-count 2 --enable-oidc-issuer --enable-workload-identity \
  --enable-ai-toolchain-operator --generate-ssh-keys -o none
az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubectl get pods -n kube-system | grep -i kaito

# (Depending on version, KAITO needs a federated identity for its node provisioner; follow the add-on docs.)

# 2. Deploy a small model with a workspace
cat <<'EOF' | kubectl apply -f -
apiVersion: kaito.sh/v1beta1
kind: Workspace
metadata: { name: workspace-phi-4-mini }
resource:
  instanceType: "Standard_NC24ads_A100_v4"
  labelSelector: { matchLabels: { apps: phi-4-mini } }
inference:
  preset: { name: phi-4-mini-instruct }
EOF
kubectl get workspace workspace-phi-4-mini -w      # wait for ResourceReady and InferenceReady

# 3. Call it through its service from inside the cluster
IP=$(kubectl get svc workspace-phi-4-mini -o jsonpath='{.spec.clusterIP}')
kubectl run curl --rm -it --image=curlimages/curl -- \
  curl -s http://$IP/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"phi-4-mini-instruct","messages":[{"role":"user","content":"What is KAITO?"}],"max_tokens":80}'

az group delete -n $RG --yes --no-wait     # GPU nodes are expensive; delete promptly
```

## Security, cost and reliability notes

- **Security:** self-hosted models have no content filter. Add Azure AI Content Safety and prompt shields at the gateway or in the app.
- **Security:** model weights are code-adjacent. Use trusted sources and safetensors, scan images, check licences, and keep inference endpoints internal and authenticated.
- **Cost:** the economics in topic 06 apply: only sustained, high utilisation makes self-hosting cheaper. Track cost per million tokens against Foundry prices.
- **Cost:** GPU pools at minimum 0 for batch; reserve the steady serving baseline.
- **Reliability:** model loading takes minutes; use startup probes with long thresholds so pods are not killed while loading.
- **Reliability:** spread serving replicas across zones; keep a fallback route to a Foundry model in the gateway.

## Common mistakes

- Building an AKS GPU platform for a workload Foundry Models could serve.
- GPU pools without taints, so ordinary pods consume expensive nodes.
- Liveness probes killing pods during model load.
- Exposing vLLM directly with no auth.
- Pulling 30 GB images on every scale-out.
- No queueing on shared GPU clusters, so one team's job starves others.
- Running the only serving replica on Spot.

## Check yourself

1. How does a pod request a GPU on AKS?
2. What does a KAITO workspace do when applied?
3. When would you use MIG instead of whole GPUs?
4. Why use Kueue or Volcano on a GPU cluster?
5. What belongs in front of self-hosted model endpoints, and why?
6. Which probe setting prevents a loading model server from being killed?
7. When is Ray the right tool?

<details>
<summary>Answers</summary>

1. By setting `resources.limits` for `nvidia.com/gpu` (exposed by the NVIDIA device plugin), with a toleration for the GPU pool taint.
2. It provisions suitable GPU nodes, obtains the model, runs an inference engine (such as vLLM) with an OpenAI-compatible API, and exposes it as a service.
3. When several small models or workloads each need only part of a large GPU and you want hardware isolation between them.
4. To queue jobs, enforce fair sharing and quotas between teams, and gang-schedule distributed jobs so partial allocations do not deadlock GPUs.
5. An AI gateway (for example API Management) for authentication, token quotas, logging, routing, fallback to managed models and content safety.
6. A startup probe with a long enough failure threshold and period, so liveness checks start only after the model has loaded.
7. When Python workloads (training, tuning, batch inference, serving) must scale across many nodes.

</details>

## References

- [Use GPUs on AKS](https://learn.microsoft.com/azure/aks/gpu-cluster)
- [Deploy AI models with the AI toolchain operator (KAITO)](https://learn.microsoft.com/azure/aks/ai-toolchain-operator)
- [KAITO project](https://github.com/kaito-project/kaito)
- [Multi-instance GPU node pools](https://learn.microsoft.com/azure/aks/gpu-multi-instance)
- [NVIDIA GPU Operator on AKS](https://learn.microsoft.com/azure/aks/nvidia-gpu-operator)
- [Ray on AKS with KubeRay](https://learn.microsoft.com/azure/aks/ray-overview)
- [Monitor GPU metrics on AKS](https://learn.microsoft.com/azure/aks/monitor-gpu-metrics)
- [AI gateway in Azure API Management](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
