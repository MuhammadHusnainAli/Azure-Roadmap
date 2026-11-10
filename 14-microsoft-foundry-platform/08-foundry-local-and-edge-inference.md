# Foundry Local and Edge Inference

## Why it matters

Not every AI call should go to the cloud. Field engineers work offline, clinicians handle data that must stay on the device, factory PCs need millisecond responses, and some features run millions of times a day where per-token costs add up. **Foundry Local** runs optimised models on the user's own device (Windows, macOS, Linux) with automatic hardware acceleration and an OpenAI-compatible API. Combined with cloud Foundry, it enables **hybrid** designs that route each request to the right place.

## Key concepts

### What Foundry Local is

Microsoft describes Foundry Local as "an end-to-end local AI solution for shipping applications that run entirely on your device". It isn't a server product.

| Feature | Detail (checked 2026-11) |
|---|---|
| **Runtime** | Handles model download, caching, hardware acceleration and inference through **ONNX Runtime**; adds about 20 MB to an app package |
| **SDKs** | C#, C++, JavaScript, Rust and Python; in-process inference is the recommended integration |
| **Curated catalogue** | Quantised, compressed models tested on consumer hardware, for example gpt-oss, Qwen, DeepSeek, Mistral and Phi for chat, and Whisper for transcription. Models are versioned, so apps can pin or auto-update |
| **Hardware acceleration** | Detects the device and selects the best execution provider: NPU, then GPU, falling back to CPU; drivers and execution providers managed automatically |
| **OpenAI-compatible API** | Supports OpenAI request and response formats, including the Responses API format |
| **Optional local server and CLI** | An OpenAI-compatible web server and CLI for development, multi-process use or tools such as LangChain |
| **Platforms** | Windows, macOS (Apple silicon), Linux |
| **Azure subscription** | Not required; no per-token costs |
| **Status** | Generally available since April 2026 (some CLI or feature pages may still say preview) |

**Privacy:** prompts and outputs are processed on the device. Network access is used for model and component downloads and optional diagnostics the user chooses to share.

**Not for servers.** Foundry Local is optimised for single-user devices. For multi-user serving, use vLLM, Triton, KAITO on AKS or Foundry managed compute. For enterprise on-premises inference with Kubernetes and Arc management, see **Foundry Local on Azure Local** (sovereign and private cloud scenarios).

### Related Microsoft on-device options

| Option | Use |
|---|---|
| **Windows AI APIs** (on Copilot+ PCs) | Built-in OS models (for example Phi Silica, imaging and OCR APIs) for Windows apps |
| **Windows ML** | Run your own ONNX models with hardware acceleration across Windows devices |
| **Foundry Local** | Cross-platform app-embedded runtime with a curated LLM catalogue and OpenAI-compatible API |
| **Azure IoT Operations / Arc-enabled edge** | Industrial edge workloads with Kubernetes (topic 31) |

### Edge and hybrid patterns

| Pattern | How it works | Example |
|---|---|---|
| **Local-only** | All inference on device | Offline note summarisation for field engineers |
| **Local-first with cloud fallback** | Try the local model; escalate to cloud for hard requests or when confidence is low | Customer-service desktop app: local intent classification, cloud for complex answers |
| **Split by sensitivity** | Sensitive data handled locally (redact, classify); only sanitised content goes to the cloud | Healthcare app redacts PHI locally before cloud summarisation |
| **Cloud-first with offline mode** | Cloud normally; local model when disconnected | Retail store app during network outages |
| **Same code, two endpoints** | OpenAI-compatible API means one client library with a switchable base URL | Dev/test locally, production in Foundry |

## How it works on Azure

### Install and try (CLI for development)

```bash
# Windows
winget install Microsoft.FoundryLocal
# macOS
brew tap microsoft/foundrylocal && brew install foundrylocal

foundry model list                # catalogue entries that fit your hardware
foundry model run phi-4-mini      # download (first time) and chat interactively
foundry service status            # local server endpoint, if the service is running
foundry cache list                # downloaded models
```

Model aliases (for example `phi-4-mini`, `qwen2.5-0.5b`, `gpt-oss-20b`) map to the best variant for your hardware (NPU, GPU or CPU builds). Check `foundry model list`, because aliases change with the catalogue.

### Call through the OpenAI SDK (local server)

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:<port>/v1", api_key="local")   # port from `foundry service status`
r = client.chat.completions.create(
    model="phi-4-mini",
    messages=[{"role": "user", "content": "Classify the ticket: 'VPN drops every hour' → network|hardware|software"}])
print(r.choices[0].message.content)
```

### Embed with the SDK (in-process, recommended for apps)

The Foundry Local SDK starts the runtime inside your app, downloads or loads the model, and gives you an OpenAI-compatible client or native chat API without a separate server. Package names and method names vary by language (for example a `foundry-local-sdk` package for Python). Follow the "Get started" guide for your language.

### Hybrid routing sketch

```python
def answer(question: str) -> str:
    local = local_client.chat.completions.create(model="phi-4-mini", messages=[
        {"role": "system", "content": "Answer briefly. If unsure or the task needs external knowledge, reply exactly: ESCALATE"},
        {"role": "user", "content": question}]).choices[0].message.content
    if local.strip() != "ESCALATE":
        return local
    # cloud: Foundry project (keyless, via the app's backend; never ship cloud credentials in a client app)
    return call_backend_cloud_agent(question)
```

In client applications, the cloud call should go through **your backend API** (authenticated with the user's Entra token), not directly to Foundry with embedded credentials.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Local vs cloud | Local for privacy, offline, latency or high-volume simple tasks; cloud for complex reasoning, large context, current knowledge, tools and governance |
| Model size on device | Smaller models (1–4B parameters, quantised) fit broadly; larger ones need capable GPUs or NPUs and more storage. Test on the lowest-spec target device |
| SDK vs local server | SDK (in-process) for shipped apps; server for development, prototyping and multi-process tooling |
| Updates | Pin model versions for predictability; allow auto-updates where you have evaluation coverage |
| Governance | On-device inference bypasses cloud content filters and logging. Add local safety checks where needed and decide what telemetry (if any) to collect with consent |

## Hands-on

1. Install Foundry Local, list models, and run `phi-4-mini`. Note the execution provider selected (NPU, GPU or CPU) and the tokens per second.
2. Call the local endpoint with the OpenAI SDK, then change `base_url` to your Foundry v1 endpoint (with Entra auth) and run the same code.
3. Disconnect from the network and confirm local inference still works.
4. Build the hybrid router above with an evaluation set of 30 questions; measure how many stay local and the quality of each path.
5. Transcribe a short audio clip with a local Whisper model.

## Security, cost and reliability notes

- **Security:** models and prompts live on the device, so protect the device (disk encryption, managed devices with Intune). Don't put secrets or cloud credentials in client apps.
- **Security:** cloud-side guardrails (content filters, Prompt Shields, logging) don't apply locally. Add appropriate local safeguards for your use case.
- **Licensing:** check each model's licence for redistribution inside your application.
- **Cost:** no per-token cost, but device requirements, app size and support for heterogeneous hardware are real costs.
- **Reliability:** first-run downloads can be large. Plan for pre-caching, bandwidth limits and graceful handling when a device lacks resources.

## Common mistakes

- Using Foundry Local as a multi-user server.
- Shipping cloud keys in a desktop or mobile app for the "fallback" path.
- Testing only on high-end developer laptops.
- Assuming local models have the same quality, context length or safety behaviour as cloud models.
- Ignoring model licences for redistribution.

## Check yourself

1. What is Foundry Local, and does it need an Azure subscription?
2. Which platforms and hardware accelerators does it support?
3. Why isn't Foundry Local suitable as a server?
4. Name three hybrid cloud and local patterns.
5. How do you switch existing OpenAI SDK code to Foundry Local?
6. What protections do you lose when inference runs locally?

<details>
<summary>Answers</summary>

1. An end-to-end on-device AI runtime and SDK (with a curated model catalogue, hardware acceleration and an OpenAI-compatible API) that ships inside your app; no Azure subscription is required.
2. Windows, macOS (Apple silicon) and Linux; NPUs and GPUs when available, falling back to CPU, through ONNX Runtime execution providers.
3. It's designed for single-user devices and lacks multi-user features such as request queuing, continuous batching and GPU sharing; use vLLM, Triton or managed compute for servers.
4. Local-first with cloud fallback, split by sensitivity (redact locally, process in cloud), cloud-first with offline mode (also local-only and same code with two endpoints).
5. Point `base_url` at the local endpoint (or use the SDK's client) and use a local model alias as `model`.
6. Cloud content filters and guardrails, centralised logging and monitoring, and cloud governance such as the AI gateway and Control Plane.

</details>

## References

- [What is Foundry Local?](https://learn.microsoft.com/azure/foundry-local/what-is-foundry-local)
- [Get started with Foundry Local](https://learn.microsoft.com/azure/foundry-local/get-started)
- [Foundry Local architecture](https://learn.microsoft.com/azure/foundry-local/concepts/foundry-local-architecture)
- [Foundry Local CLI reference](https://learn.microsoft.com/azure/foundry-local/reference/reference-cli)
- [Foundry Local on Azure Local](https://learn.microsoft.com/azure/azure-sovereign-clouds/private/foundry-local/what-is-foundry-local-on-azure-local)
- [Windows AI on Windows](https://learn.microsoft.com/windows/ai/)
