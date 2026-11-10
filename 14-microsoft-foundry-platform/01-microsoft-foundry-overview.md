# Microsoft Foundry Overview

## Why it matters

**Microsoft Foundry** is Azure's platform for building, deploying and governing AI applications and agents. Model access, agent hosting, knowledge (Foundry IQ), prebuilt AI tools, evaluation, observability and fleet governance all sit under one resource, one portal and one SDK. Almost every AI engagement on Azure now starts here. Because the product has been renamed several times, you need to recognise old names in existing estates and documentation and use the current names in new work.

## Key concepts

### What Foundry is

Microsoft describes Foundry as a unified platform for enterprise AI operations, model builders and application development. In practical terms it combines:

| Capability | What you get | Covered in |
|---|---|---|
| **Foundry Models** | A catalogue of thousands of models (Azure OpenAI, Anthropic, Microsoft MAI and Phi, Meta, Mistral, xAI, DeepSeek, Cohere, open models), deployments, model router | Topic 15 |
| **Foundry Agent Service** | Hosting and runtime for prompt, workflow and hosted agents, with tools, memory and publishing to Teams and Microsoft 365 Copilot | Topic 18 |
| **Foundry IQ** | Knowledge bases over Azure AI Search for agentic retrieval | Topic 17 |
| **Foundry Tools** | Prebuilt AI services: Speech, Language, Translator, Vision, Content Understanding, Document Intelligence, Content Safety | Topic 20 |
| **Evaluation and observability** | Evaluators, red teaming, tracing, monitoring | Topic 22 |
| **Guardrails** | Content filters and guardrail policies on inputs, outputs, tool calls | Topic 23 |
| **Foundry Control Plane** | Fleet-wide inventory, compliance, monitoring and security for agents, models and tools | File 09 |
| **Developer tooling** | Foundry SDK, OpenAI-compatible endpoints, Foundry Toolkit for VS Code, azd templates, Foundry Local | Files 06–08 |

### The rename history

| Period | Name | Notes |
|---|---|---|
| 2023–2024 | **Azure AI Studio** | Portal on top of hubs and projects (Azure Machine Learning workspaces) |
| Nov 2024 – 2025 | **Azure AI Foundry** | Rename at Ignite 2024; introduced the Foundry resource and Foundry projects in 2025 |
| Late 2025 onwards | **Microsoft Foundry** | Current name. The "new" Foundry portal and APIs; the older experience is called **Foundry (classic)** |

Related renames you'll meet:

| Old | Current |
|---|---|
| Azure AI services / Cognitive Services | **Foundry Tools** (individual services keep names such as Azure Speech in Foundry Tools) |
| Azure OpenAI Service | **Azure OpenAI in Foundry Models** |
| Azure AI Agent Service | **Foundry Agent Service** |
| Azure AI Model Inference | Part of **Foundry Models** |
| AI Toolkit for VS Code | **Foundry Toolkit for Visual Studio Code** |
| Azure AI User / Owner / Account Owner / Project Manager roles | **Foundry User / Owner / Account Owner / Project Manager** (same role IDs) |

Documentation lives under `learn.microsoft.com/azure/foundry/` (new) and `learn.microsoft.com/azure/foundry-classic/` (classic, hub-based). Older links under `/azure/ai-foundry/` and `/azure/ai-studio/` redirect.

### Foundry vs Foundry (classic)

| | Microsoft Foundry (new) | Foundry (classic) |
|---|---|---|
| Resource model | **Foundry resource** (`Microsoft.CognitiveServices/accounts`, kind `AIServices`) with child **projects** | Hub (`Microsoft.MachineLearningServices/workspaces`, kind `Hub`) with hub-based projects; or Foundry projects viewed in the classic portal |
| Portal | Foundry portal at `ai.azure.com` (new experience) | Classic experience |
| Agents | Agent Service with prompt, workflow and hosted agents, Responses API-based | Earlier Agents (Assistants-style threads and runs) |
| SDK | `azure-ai-projects` 2.x | `azure-ai-projects` 1.x |
| When to use | All new work | Existing hub-based estates; features not yet in Foundry projects (check the feature comparison, for example prompt flow) |

### Portal tour (new experience)

| Area | What you do there |
|---|---|
| **Home / project overview** | Project endpoint, keys (if enabled), quick starts |
| **Discover / model catalogue** | Browse models, benchmarks and leaderboards, compare, deploy |
| **Build** | Agents, workflows, playgrounds (chat, agents, images, audio), knowledge (Foundry IQ), tools and toolboxes, fine-tuning, evaluations |
| **Operate** (Control Plane) | Fleet overview, assets (agents, models, tools), compliance (guardrail policies), monitoring and alerts |
| **Manage** | Project details, resource details, users and roles, connected resources, quota, AI gateway |

Portal navigation changes frequently. Use the "Navigate from classic" guide if you know the old layout.

## How it works on Azure

```text
Subscription
 └─ Resource group
     └─ Foundry resource  (Microsoft.CognitiveServices/accounts, kind=AIServices)
          ├─ Model deployments (shared by all projects)
          ├─ Networking, encryption, guardrails, connections (resource level)
          ├─ Project A (accounts/projects) ── agents, files, evaluations, project connections, managed identity
          └─ Project B
     Connected resources (separate governance): Storage, Key Vault, AI Search, Cosmos DB, App Insights, APIM
```

Endpoints of a Foundry resource:

| Endpoint | Used for |
|---|---|
| `https://<resource>.services.ai.azure.com/api/projects/<project>` | **Project endpoint**: Foundry SDK, agents, evaluations, connections |
| `https://<resource>.openai.azure.com/openai/v1` | OpenAI-compatible endpoint (OpenAI SDK): Chat Completions, Responses, embeddings, images, audio |
| `https://<resource>.services.ai.azure.com/anthropic` | Anthropic SDK for Claude models deployed in Foundry |
| `https://<resource>.cognitiveservices.azure.com/` | Foundry Tools APIs (Speech, Language, Vision, Translator...) |

### First call in two minutes

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project = AIProjectClient(
    endpoint="https://<resource>.services.ai.azure.com/api/projects/<project>",
    credential=DefaultAzureCredential())

openai = project.get_openai_client()
r = openai.responses.create(model="<deployment-name>", input="In one sentence, what is Microsoft Foundry?")
print(r.output_text)
```

`pip install "azure-ai-projects>=2.3.0" "openai>=3.0.0"`. You need the **Foundry User** role on the project or resource.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Foundry vs standalone Azure OpenAI resource | Foundry resource for anything beyond simple completions: agents, evaluations, multiple model providers, projects. A standalone Azure OpenAI resource only for simple, OpenAI-only workloads (it exposes only `/openai/v1`) |
| New vs classic | New Foundry for all new projects; plan migration for hub-based projects unless they depend on classic-only features |
| Foundry vs Copilot Studio | Foundry for pro-code, custom apps, external users, rigorous evaluation and networking; Copilot Studio for low-code Microsoft 365-centric agents (topic 24) |
| Foundry vs Azure Machine Learning | Foundry for generative AI apps and agents; Azure ML for custom model training and MLOps (topic 21) |

## Hands-on

1. Open `ai.azure.com`, create a Foundry resource and project (portal quick start), and note the project endpoint.
2. Deploy a small chat model from the catalogue and try it in the playground.
3. Run the Python "first call" sample with `DefaultAzureCredential` after `az login`.
4. Browse **Operate** and **Manage** and list what each area shows.
5. If you have an old hub-based project, open it in the classic experience and compare the resource types in the Azure portal.

## Security, cost and reliability notes

- **Security:** prefer Microsoft Entra ID (keyless) authentication and disable key auth where possible (file 03). Roles were renamed, so use role IDs in automation.
- **Security:** connected resources (Storage, Search, Key Vault) are separate governance boundaries. Secure their networking and RBAC too.
- **Cost:** the Foundry resource itself has no standing charge. You pay for model usage, agent tools, search, storage, evaluations and compute you use (file 10).
- **Reliability:** Foundry doesn't fail over automatically across regions. For multi-region availability, deploy separate resources and route at the application or gateway layer.

## Common mistakes

- Using outdated names (Azure AI Studio, Azure OpenAI Service, Azure AI services) in new designs and proposals.
- Starting new work on hubs because an old tutorial used them.
- Assuming the Foundry resource protects connected resources' networking.
- Assigning legacy "Cognitive Services" roles for Foundry project work.

## Check yourself

1. List three earlier names of Microsoft Foundry.
2. What Azure resource type and kind is a Foundry resource?
3. What is the project endpoint format?
4. What are Foundry Tools?
5. When might you still use a hub-based project?
6. Which SDK version targets the new Foundry?

<details>
<summary>Answers</summary>

1. Azure AI Studio, Azure AI Foundry (and the classic Foundry experience).
2. `Microsoft.CognitiveServices/accounts` with kind `AIServices`; projects are `accounts/projects` sub-resources.
3. `https://<resource-name>.services.ai.azure.com/api/projects/<project-name>`.
4. The prebuilt AI services formerly called Azure AI services or Cognitive Services, such as Speech, Language, Translator, Vision, Content Understanding, Document Intelligence and Content Safety.
5. When you rely on features not yet available in Foundry projects (check the feature comparison, for example prompt flow), or for existing estates not yet migrated.
6. `azure-ai-projects` 2.x (Python `>=2.3.0`); 1.x targets Foundry classic.

</details>

## References

- [What is Microsoft Foundry?](https://learn.microsoft.com/azure/foundry/what-is-foundry)
- [Microsoft Foundry architecture](https://learn.microsoft.com/azure/foundry/concepts/architecture)
- [Get started with Foundry SDKs and endpoints](https://learn.microsoft.com/azure/foundry/how-to/develop/sdk-overview)
- [Navigate from Foundry (classic)](https://learn.microsoft.com/azure/foundry/how-to/navigate-from-classic)
- [Foundry (classic) documentation](https://learn.microsoft.com/azure/foundry-classic/what-is-foundry)
- [Foundry rollout planning](https://learn.microsoft.com/azure/foundry/concepts/planning)
