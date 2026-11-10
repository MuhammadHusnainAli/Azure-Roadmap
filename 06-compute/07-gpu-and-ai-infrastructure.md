# GPU and AI Infrastructure

## Why it matters

Most AI applications on Azure call a managed model endpoint and never touch a GPU. Some need their own: an open-weight or custom-trained model, strict data or sovereignty constraints, specialised inference, or training and fine-tuning beyond what managed services offer. Knowing what GPU infrastructure involves, and what it costs in money and operational effort, lets an engineer build it properly and lets an architect or consultant answer the question "should we host our own model?" with numbers instead of instinct. GPU families and availability change fast; verify specifics before committing.

## Key concepts

### The first question: do you need GPUs at all?

| Option | You manage | Billing | Typical fit |
|---|---|---|---|
| **Foundry Models, standard (serverless) deployments** | Prompts, access, quota | Per token | The default. Frontier and many open models with no infrastructure |
| **Foundry Models, provisioned throughput** | Capacity sizing | Per provisioned unit per hour | Steady high volume needing predictable latency |
| **Foundry managed compute / Azure Machine Learning managed online endpoints** | Model choice, instance type and count, scaling rules | Per VM hour | Open or custom models with a managed serving layer |
| **Azure Container Apps serverless GPUs** | Container image, scaling rules | Per second of GPU use, scale to zero | Bursty or intermittent custom inference |
| **AKS with GPU node pools** (optionally KAITO) | Cluster, nodes, serving stack | Per VM hour | Kubernetes-standard teams, many models, fine control |
| **GPU virtual machines / scale sets** | Everything | Per VM hour | Full control, specialised setups, training clusters |
| **Azure Batch / CycleCloud** | Job definitions, pools | Per VM hour | Batch inference, large training and HPC-style jobs |
| **Foundry Local / on-device** | The device | None in the cloud | Offline, edge, privacy |

Work down the list and stop at the first that meets the requirement. Each step down adds control and adds operational burden.

### When self-hosting is justified

| Driver | Detail |
|---|---|
| **The model is not offered as a service** | A specific open-weight model, your own trained model, or a heavily customised one |
| **Data or sovereignty constraints** | Policy requires the model to run in infrastructure you control, in a specific location, possibly disconnected |
| **Cost at sustained high volume** | A fully used GPU can be cheaper per token than an API. The condition is *fully used* |
| **Latency and control** | Custom batching, quantisation, co-location with data, guaranteed capacity |
| **Customisation beyond what APIs allow** | Custom decoding, adapters, architectures |
| **Confidential computing** | Running in an attested TEE (file 06) |

### When it is not

- "It will be cheaper" without a utilisation estimate. A GPU at 15% use is far more expensive per token than any API.
- To access frontier proprietary models. You cannot self-host them.
- Without a team to run it: drivers, serving stack, scaling, patching, monitoring, capacity, model updates, on-call.
- For a prototype. Start on an API; revisit with real usage data.

### The economics

The comparison is **cost per useful token** (or per request), not price per hour.

```text
Self-hosted cost per 1M tokens  =  (GPU VM hourly price × hours)  ÷  (tokens actually served, in millions)

API cost per 1M tokens          =  list price (input and output priced separately)
```

What determines throughput on a GPU: the model's size and precision (quantisation), the serving engine and batching, context lengths, concurrency, and the mix of input and output tokens. Measure it with your own workload.

What people leave out of the self-hosted side:

| Hidden cost | Note |
|---|---|
| **Idle time** | Nights, weekends, uneven traffic. The largest factor |
| **Redundancy** | At least two instances, across zones, for availability |
| **Scale headroom** | Capacity held for peaks |
| **Engineering and operations** | People time to build and run the platform |
| **Evaluation and safety** | Content filtering, abuse monitoring and evaluations you now provide yourself |
| **Model lifecycle** | Testing and rolling out new versions |
| **Capacity risk** | Reservations to guarantee scarce GPUs |

A rough rule: self-hosting competes on cost only with **sustained, predictable, high utilisation** of a model that is good enough for the task, and with a team already capable of operating it. Otherwise the justification must be capability, control or compliance.

### GPU virtual machine families

Azure's GPU sizes are the **N** family.

| Sub-family | Purpose | Examples of hardware across generations |
|---|---|---|
| **NC** | Compute: training and inference at small to medium scale | NVIDIA T4, A10, A100, H100 |
| **ND** | Large-scale deep learning training and high-throughput inference, multi-GPU, often with **InfiniBand** | NVIDIA A100, H100, H200, and newer accelerated systems; AMD MI300X |
| **NV** | Visualisation, virtual desktops, graphics, light inference | NVIDIA A10 (with fractional GPU sizes), AMD Radeon |
| **NG** | Cloud gaming and graphics | AMD Radeon |
| **NCC / confidential** | Confidential GPU computing | NVIDIA H100 in a TEE |

Reading a name: `Standard_NC24ads_A10_v4` is NC family, 24 vCPUs, AMD CPU, local disk, premium storage, A10 GPU, version 4. `Standard_ND96isr_H100_v5` adds `i` (isolated), and `r` (RDMA, meaning InfiniBand).

Selection factors:

| Factor | Why it matters |
|---|---|
| **GPU memory (VRAM)** | The model's weights plus working memory for context and batch must fit. The first constraint for inference |
| **Number of GPUs** | Larger models and higher throughput; requires a serving stack that uses several |
| **GPU generation** | Newer GPUs deliver far more tokens per second and per unit cost for large models |
| **Fractional GPU sizes** (some NV sizes) | A slice of a GPU for light workloads |
| **InfiniBand (RDMA)** | Low-latency interconnect for multi-node distributed training |
| **Local NVMe** | Fast loading of model weights and datasets |
| **CPU and RAM** | Data loading and preprocessing can bottleneck a fast GPU |

Sizing memory for inference, as a first estimate: parameters × bytes per parameter (4 for 32-bit float, 2 for 16-bit, about 1 for 8-bit, about 0.5 for 4-bit quantisation), plus overhead for context and batching. A 7-billion-parameter model at 16-bit needs roughly 14 GB plus overhead; at 4-bit, roughly 4 to 5 GB. Quantisation trades some quality for large memory and speed gains.

Older GPU series are retired regularly (several NC and NV generations already have been). Do not build on a series with an announced retirement.

### Quota and capacity

GPU capacity is the most constrained resource in Azure.

- **Quota is per GPU family per region and is often zero by default.** Request it through *Quotas* with a justification. Approval is not automatic and depends on regional capacity.
- **Quota is not capacity.** An approved quota can still meet allocation failures.
- **Capacity varies sharply by region.** Be flexible on region, and consider where newer hardware is being deployed.
- **On-demand capacity reservations** guarantee capacity for production inference.
- **Spot** GPU capacity is cheap and volatile: usable for checkpointed training and batch inference, not for serving.
- **Reservations and savings plans** discount GPU VMs that run continuously.
- Start the quota conversation **weeks** before you need the hardware.

### The software stack

```text
Application / client
Inference server      vLLM · NVIDIA Triton · NVIDIA NIM · Text Generation Inference · Ollama · ONNX Runtime
Framework             PyTorch · TensorRT · ONNX
CUDA libraries        CUDA, cuDNN  (versions must match the driver and framework)
Container runtime     containerd + NVIDIA Container Toolkit
GPU driver            NVIDIA (CUDA) driver for NC/ND · GRID driver for NV visualisation · AMD driver for AMD GPUs
Operating system      Ubuntu / Azure Linux
VM                    N-series size
```

Ways to get drivers on: the **GPU driver VM extension**, a marketplace HPC or AI image, the **NVIDIA GPU Operator** or AKS-managed driver installation on Kubernetes, or your own golden image (file 05). In production, **pin** driver, CUDA and framework versions together and test upgrades.

Check the GPU from inside:

```bash
nvidia-smi                        # GPUs, memory, utilisation, driver and CUDA versions
nvidia-smi dmon -s u              # live utilisation
```

**Inference servers** matter as much as hardware. Engines such as vLLM provide continuous batching, efficient memory use and an OpenAI-compatible API, and can multiply throughput compared with a naive loop. Most expose `/v1/chat/completions`, so applications written against the OpenAI API need only an endpoint change.

### Hosting options compared

| | **Managed endpoints (Foundry managed compute / Azure ML)** | **Container Apps serverless GPU** | **AKS + GPU node pools** | **VMs / scale sets** |
|---|---|---|---|---|
| Effort | Low | Low | Medium to high | High |
| Scale to zero | Limited | **Yes** | With node autoscaling to zero, slower | Manual |
| Cold start | Minutes when scaling instances | Tens of seconds to minutes (model load) | Minutes for a new node | Minutes |
| GPU choice | From supported instance types | Limited set (for example T4, A100) | Any available N size | Any |
| Multi-model, custom routing | Per endpoint, traffic splitting | Per app | Full control | Full control |
| Catalogue integration | One-click deploy of catalogue models | Bring your container | KAITO presets for popular models | None |
| Best for | Open or custom models with minimal ops | Intermittent or bursty inference | Platform teams standardised on Kubernetes | Special cases, training clusters |

**KAITO** (Kubernetes AI Toolchain Operator), available as an AKS add-on, automates deploying open models on AKS: you declare a workspace naming a model preset, and it provisions suitable GPU nodes, pulls the model image and exposes an inference endpoint. It also supports fine-tuning and retrieval workloads. Topic 07 covers it.

**Foundry managed compute** deploys catalogue models to dedicated VMs behind a managed endpoint, addressing cases where a model is not offered serverlessly or you need dedicated capacity. Check its current status and regional availability.

### Training and fine-tuning infrastructure

| Scale | Approach |
|---|---|
| **Fine-tuning a hosted model** | Foundry fine-tuning as a service: no infrastructure (topic 16) |
| **Single-GPU or single-node fine-tuning** (adapters, small models) | Azure Machine Learning compute, a GPU VM, or a Container Apps or AKS job |
| **Multi-GPU, single node** | Larger NC or ND sizes |
| **Distributed multi-node training** | ND sizes with InfiniBand; orchestrated by Azure Machine Learning, AKS with a training operator, Batch, or CycleCloud with Slurm |
| **Data** | Blob or Data Lake, or Azure Managed Lustre for high-throughput shared access |
| **Experiment tracking** | Azure Machine Learning with MLflow |

Training practices:

- **Checkpoint** to Blob Storage frequently; it enables Spot and protects long runs.
- **Use Spot** for cost where interruptions are tolerable.
- Keep **GPU utilisation high**: profile data loading, which often starves the GPU.
- **Mixed precision** and efficient fine-tuning methods (LoRA and relatives) reduce hardware needs dramatically.
- Co-locate compute and data in the same region.
- Shut clusters down when idle. Autoscale to zero.

Most organisations will fine-tune rarely and train from scratch never. The common pattern is: managed model APIs for inference, occasional managed fine-tuning, and no standing GPU fleet.

## How it works on Azure

### A production self-hosted inference design

```text
Clients → API Management (auth, token limits, routing, logging)
              │
              ├─► Foundry Models (managed)                 ← default path, most traffic
              └─► Self-hosted model endpoint               ← where justified
                     AKS GPU node pool, zones 1·2·3 (or managed endpoint / Container Apps)
                       • vLLM / Triton behind an internal service, OpenAI-compatible API
                       • min 2 replicas; autoscale on queue depth or GPU utilisation
                       • model weights from Blob / registry to local NVMe at start
                       • capacity reservation for the base; optional Spot for burst
                       • golden node image with pinned driver + CUDA
                     Private networking; managed identity; no public endpoint
              Observability: tokens/sec, time to first token, queue length, GPU utilisation and memory, error rate
              Safety: Azure AI Content Safety called by the gateway or app, since the model has no built-in filter
```

Points that self-hosting makes yours:

| Concern | With a managed API | Self-hosted |
|---|---|---|
| Content filtering and prompt shields | Built in | Call Azure AI Content Safety yourself |
| Abuse monitoring | Provided | Yours |
| Scaling and capacity | Quota request | Capacity planning, autoscale, reservations |
| Model updates | Version upgrade | Download, validate, roll out |
| Evaluation | Same either way | Same, plus regression tests on each model change |
| Licence compliance | Covered by the service terms | Read the model's licence; some restrict commercial use |
| Security patching | Provider | Drivers, runtime, OS, serving stack |

Putting both behind the same gateway with an OpenAI-compatible interface keeps the application indifferent to which model serves a request, and lets you move traffic as economics change.

### Monitoring a GPU workload

| Metric | Why |
|---|---|
| **GPU utilisation and memory** | Are you getting value from the hardware? Out-of-memory risk |
| **Tokens per second, requests per second** | Throughput |
| **Time to first token, latency percentiles** | User experience |
| **Queue length / pending requests** | The best autoscale signal for inference |
| **Batch size achieved** | Efficiency of the serving engine |
| **Node and pod restarts, Xid errors** | Hardware and driver health |
| **Cost per million tokens (derived)** | The number to compare with the API |

Azure Monitor collects GPU metrics from VMs (through the agent and NVIDIA's exporter) and from AKS (managed Prometheus with the DCGM exporter).

### Capacity planning steps

1. **Characterise the workload:** requests per second at peak, input and output tokens per request, latency target.
2. **Benchmark** the candidate model on a candidate GPU with a realistic prompt mix and concurrency. Record tokens per second at your latency target.
3. **Compute instances needed** for peak, add redundancy (N+1 across zones) and headroom.
4. **Estimate utilisation** across a week, not only at peak.
5. **Price it:** VM cost for that fleet (with reservations where appropriate), plus storage, networking and people.
6. **Compare** with the managed API cost for the same token volume, including provisioned throughput.
7. **Check quota and capacity** in the target region before deciding.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Managed API or self-hosted | Managed API unless a specific driver from the list above applies and the utilisation and team exist |
| Which hosting option | Highest abstraction that works: managed endpoints or Container Apps before AKS before raw VMs |
| Which GPU | Smallest that fits the model with headroom, on a current generation. Benchmark two options |
| Quantisation | Usually worthwhile for inference. Validate quality on your own evaluation set |
| One large model or several small | Small specialised models are far cheaper to host. Route tasks to the smallest adequate model |
| Scale to zero | Good for intermittent use if cold starts are acceptable. Keep a warm minimum for interactive use |
| Region | Follow capacity and data constraints. Private backbone latency to another region is usually acceptable |
| Hybrid | Managed APIs for frontier quality, self-hosted for volume or sensitive subsets, behind one gateway |
| Build training infrastructure | Rarely. Prefer managed fine-tuning and efficient methods |

## Hands-on

GPU VMs are expensive and often unavailable to new subscriptions. Steps 1 to 3 are free.

```bash
LOC=swedencentral

# 1. What GPU sizes exist in the region, and are they restricted for you?
az vm list-skus -l $LOC --resource-type virtualMachines \
  --query "[?starts_with(name,'Standard_N')].{Size:name, vCPU:capabilities[?name=='vCPUs'].value|[0], MemGB:capabilities[?name=='MemoryGB'].value|[0], GPUs:capabilities[?name=='GPUs'].value|[0], Zones:join(',',locationInfo[0].zones), Restricted:restrictions[0].reasonCode}" -o table

# 2. Your GPU quota (expect zeros)
az vm list-usage -l $LOC \
  --query "[?contains(name.value,'NC') || contains(name.value,'ND') || contains(name.value,'NV')].{Family:name.localizedValue, Used:currentValue, Limit:limit}" -o table

# 3. Prices: on-demand vs Spot for a GPU size
SIZE=Standard_NC4as_T4_v3
curl -s "https://prices.azure.com/api/retail/prices?\$filter=armRegionName eq '$LOC' and armSkuName eq '$SIZE' and priceType eq 'Consumption'" \
  | jq -r '.Items[] | select(.productName|test("Windows")|not) | [.skuName, .unitPrice] | @tsv'
```

**Economics exercise (do this even without a GPU).** Using the prices from step 3 and a current managed model price:

1. Assume one GPU instance serves 1,500 output tokens per second at full load for a small open model.
2. Compute cost per million output tokens at 100%, 50% and 10% utilisation.
3. Double it for two instances across zones.
4. Compare with the API price for a comparable small model. At what utilisation does self-hosting break even? What has the calculation left out?

**If you have quota: run a model on a GPU VM**

```bash
RG=rg-gpu-lab; az group create -n $RG -l $LOC -o none
az vm create -g $RG -n vm-gpu --size Standard_NC4as_T4_v3 --image Ubuntu2204 \
  --public-ip-address "" --admin-username azureuser --generate-ssh-keys \
  --os-disk-size-gb 128 -o none
# (give the subnet a NAT gateway so the VM can download drivers and the model)

# NVIDIA driver through the VM extension
az vm extension set -g $RG --vm-name vm-gpu --publisher Microsoft.HpcCompute --name NvidiaGpuDriverLinux -o none

az vm run-command invoke -g $RG -n vm-gpu --command-id RunShellScript --scripts '
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv
curl -fsSL https://ollama.com/install.sh | sh >/dev/null 2>&1
ollama pull phi4-mini >/dev/null 2>&1
curl -s http://localhost:11434/v1/chat/completions -H "Content-Type: application/json" \
  -d "{\"model\":\"phi4-mini\",\"messages\":[{\"role\":\"user\",\"content\":\"Explain a private endpoint in one sentence.\"}]}" | head -c 600
nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv' --query "value[0].message" -o tsv

az group delete -n $RG --yes --no-wait        # do not leave a GPU VM running
```

Model names and installer URLs change; substitute a current small model. Note that the endpoint is OpenAI-compatible, so the application code from topic 04 would work against it with only a base URL change.

**Lower-cost alternative:** deploy a small open model to **Azure Container Apps with a serverless GPU** workload profile, or deploy a catalogue model to a **managed endpoint** from the Foundry portal, call it, and delete it.

## Security, cost and reliability notes

- **Security:** a self-hosted model has no built-in content filtering, prompt shields or abuse monitoring. Add Azure AI Content Safety and your own controls.
- **Security:** model files and container images are supply-chain artefacts. Use trusted sources, verify checksums, scan images, prefer safe weight formats (such as safetensors) over pickled files that can execute code, and check licences.
- **Security:** keep the endpoint private and authenticated. An open inference endpoint will be found and used by others.
- **Cost:** GPU VMs are among the most expensive resources in Azure, and idle ones cost the same as busy ones. Set budgets and alerts, autoscale to zero where possible, and delete lab resources immediately.
- **Cost:** track cost per million tokens against the managed API continuously, not once.
- **Reliability:** capacity is the principal risk. Reserve what production needs and have a fallback path to a managed model.
- **Reliability:** driver and framework mismatches cause opaque failures. Pin and test.
- **Reliability:** model loading takes time. Account for it in scale-out and health probes, or new replicas receive traffic before they can serve.

## Common mistakes

- Self-hosting to save money without measuring utilisation.
- Forgetting redundancy, idle time and people in the cost model.
- Assuming GPU quota and capacity exist in the chosen region.
- Raw VMs where a managed endpoint or serverless GPU would do.
- A naive serving loop with no batching, wasting most of the GPU.
- Unpinned driver, CUDA and framework versions.
- No content safety layer.
- A public, unauthenticated inference endpoint.
- Building on a GPU series scheduled for retirement.
- A training cluster left running between jobs.
- A model whose licence does not permit the intended use.
- Hosting a large general model where a small specialised one would serve.

## Check yourself

1. List the hosting options for model inference from most to least managed.
2. Give three legitimate reasons to self-host a model and two poor ones.
3. What is the correct unit for comparing self-hosted and API costs, and what is the dominant hidden factor?
4. What do the NC, ND and NV sub-families target?
5. What is the first hardware constraint when choosing a GPU for inference, and how does quantisation affect it?
6. Why is "quota approved" not the same as "can deploy"?
7. What must you add yourself when serving a self-hosted model that a managed API provides?
8. What is KAITO?
9. Why put self-hosted and managed models behind the same gateway?

<details>
<summary>Answers</summary>

1. Foundry Models serverless, provisioned throughput, managed compute or Azure ML managed endpoints, Container Apps serverless GPUs, AKS GPU node pools, GPU VMs and scale sets (with Batch or CycleCloud for jobs).
2. Legitimate: the model is not offered as a service; data, sovereignty or confidentiality constraints; sustained high utilisation making it cheaper; required customisation or control. Poor: assuming it is cheaper without utilisation data; wanting a frontier proprietary model; doing it for a prototype.
3. Cost per token (or request) actually served. Idle time, meaning utilisation.
4. NC: general GPU compute for training and inference. ND: large-scale deep learning training and high-throughput inference, often with InfiniBand. NV: visualisation and virtual desktop graphics.
5. GPU memory: the model must fit with working overhead. Quantisation reduces bytes per parameter, cutting memory needs substantially at some cost in quality.
6. Quota is permission to request; capacity is whether hardware is physically available at that moment in that region.
7. Content filtering and prompt shields, abuse monitoring, scaling and capacity planning, patching, model lifecycle management, and licence compliance checks.
8. The Kubernetes AI Toolchain Operator: an AKS add-on that automates provisioning GPU nodes and deploying open models as inference (or fine-tuning) workloads from a declarative workspace.
9. Applications stay unchanged while traffic is routed, limited, logged and shifted between back ends as cost, capacity and quality change.

</details>

## References

- [GPU-accelerated virtual machine sizes](https://learn.microsoft.com/azure/virtual-machines/sizes/overview#gpu-accelerated)
- [NVIDIA GPU Driver Extension for Linux](https://learn.microsoft.com/azure/virtual-machines/extensions/hpccompute-gpu-linux)
- [Use GPUs for compute-intensive workloads on AKS](https://learn.microsoft.com/azure/aks/gpu-cluster)
- [Deploy an AI model on AKS with the AI toolchain operator (KAITO)](https://learn.microsoft.com/azure/aks/ai-toolchain-operator)
- [Using serverless GPUs in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/gpu-serverless-overview)
- [Deployment options for Foundry Models](https://learn.microsoft.com/azure/foundry/concepts/deployments-overview)
- [Online endpoints and deployments for real-time inference (Azure Machine Learning)](https://learn.microsoft.com/azure/machine-learning/concept-endpoints-online)
- [AI workloads on Azure infrastructure (Well-Architected Framework)](https://learn.microsoft.com/azure/well-architected/ai/application-platform)
- [AI on IaaS (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/infrastructure/well-architected)
- [Retired and retiring GPU VM sizes](https://learn.microsoft.com/azure/virtual-machines/sizes/retirement/retired-sizes-list)
