# Legacy AI Stack

## Why it matters

Generative AI on Azure is only a few years old, yet it already has a legacy layer. Systems built in 2023–2025 typically use:
- The Azure OpenAI **Assistants API**.
- **Hub-based** Azure AI Studio or Azure AI Foundry projects.
- **Prompt flow** pipelines.
- **Semantic Kernel** agents and **AutoGen** multi-agent prototypes.
- Standalone Azure OpenAI resources with API keys.
- Chat Completions code pinned to model versions that are retiring.

When you inherit one of these systems you need to know what still works, what's retired, what to migrate to, and in what order. This file is a field guide for that situation.

## Key concepts

### Status summary

| Component | Status (at time of writing) | Successor |
|---|---|---|
| **Assistants API** (threads, runs, assistants) | **Retired** | **Foundry Agent Service** with the **Responses API** (agents, conversations, responses) |
| **Classic Foundry agents** (Agent Service v1 or earlier API surface) | Legacy | Current Foundry Agent Service APIs (check the migration guide) |
| **Hub-based projects** (`Microsoft.MachineLearningServices/workspaces`, kind `Hub`) | Legacy, documented as "classic" | **Foundry resource with projects** (`Microsoft.CognitiveServices/accounts`, kind `AIServices`) |
| **Prompt flow** (Foundry and Azure Machine Learning) | **Retires 20 April 2027**; no new development | **Microsoft Agent Framework** (workflows), code-first orchestration, Foundry evaluations |
| **Foundry visual workflows** (Preview) | **Retires 1 December 2026** | Agent Framework workflows, Logic Apps |
| **Semantic Kernel v1.x** | Supported for critical bugs and security; feature development moved to Agent Framework | **Microsoft Agent Framework** |
| **AutoGen** | **Archived**; no Microsoft fixes or support | **Microsoft Agent Framework** |
| **Standalone Azure OpenAI resources** | Supported, but Foundry resources are the strategic path | Foundry resource (Azure OpenAI in Foundry Models plus other models, agents, tools) |
| **Chat Completions API** | Supported; the Responses API is recommended for new agentic work | **Responses API** |
| **Retired model versions** | Fixed retirement dates; some auto-upgrade | Current model versions after evaluation |
| **Stored completions / classic distillation** | Retired 15 October 2026 | Responses API and agent traces |

See [Topic 19 file 03](../19-agent-frameworks-and-protocols/03-migrating-from-semantic-kernel-and-autogen.md) for detailed framework migration mappings, and [Topic 14](../14-microsoft-foundry-platform/README.md) for the new and classic Foundry resource models.

### How to recognise each one in code or IaC

| Clue | Indicates |
|---|---|
| `client.beta.assistants`, `threads.runs.create`, `OpenAIAssistantAgent` | Assistants API |
| `Microsoft.MachineLearningServices/workspaces` with `kind: 'Hub'` | Hub-based project |
| `flow.dag.yaml`, `promptflow` package, `pf` CLI | Prompt flow |
| `semantic_kernel`, `Microsoft.SemanticKernel` | Semantic Kernel |
| `autogen_agentchat`, `autogen_core` | AutoGen |
| `api-key` header, `AzureOpenAI(api_key=...)` | Key-based auth (security debt) |
| `api_version="2024-...-preview"` | Old preview API versions; check retirement |
| Model deployment versions such as `2024-05-13` | Check model retirement dates |

## How it works on Azure

### Inheritance assessment (first two weeks)

1. **Inventory**: resources (Resource Graph), repos, dependencies, model deployments with versions, API versions, auth methods.
2. **Classify** each component: retired (urgent), retiring with a date, legacy but supported, current.
3. **Assess risk**: business criticality × time to retirement × migration complexity.
4. **Establish a safety net** before changing anything: an evaluation dataset and baseline metrics ([Topic 22](../22-evaluation-observability-and-genaiops/README.md)), plus tracing.
5. **Plan migration waves** with the earliest deadlines first.

### Typical migration order

| Wave | Work |
|---|---|
| 1 | Anything already retired or retiring within six months (Assistants API, visual workflows, retiring model versions) |
| 2 | Security debt: key-based auth to Entra ID; public endpoints to private |
| 3 | Prompt flow to Agent Framework or code (before April 2027) |
| 4 | Hub-based projects to Foundry projects |
| 5 | Semantic Kernel and AutoGen to Agent Framework |
| 6 | Chat Completions to the Responses API where agentic features are needed |

### Assistants API to Foundry Agent Service: concept mapping

| Assistants API | Foundry Agent Service / Responses |
|---|---|
| Assistant | Agent (versioned definition) |
| Thread | Conversation |
| Run | Response |
| File search, code interpreter tools | Equivalent built-in tools |
| Function tools | Function tools, plus OpenAPI, MCP and other tools |

Check the official migration guide for current API shapes and SDK methods. Don't port the code line by line from memory.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Migrate vs rebuild** | Rebuild small prototypes on the current stack; migrate large production systems incrementally |
| **Keep Semantic Kernel?** | Acceptable short-term for stable, non-agent features; migrate agent features first |
| **Big-bang vs strangler** | Strangler pattern: route a share of traffic to the new implementation and compare evaluation results |
| **Model upgrade alongside migration** | Avoid changing framework and model at the same time unless evaluation coverage is strong |

## Hands-on

1. Scan a repo for the clues in the table and classify the components found.
2. Build an evaluation baseline for an inherited assistant before changing it.
3. Migrate a small Assistants API sample to Foundry Agent Service and compare outputs.
4. Convert a prompt flow DAG into Agent Framework code or workflows.
5. Write a migration roadmap with dates and owners.

## Security, cost and reliability notes

- Legacy systems often use API keys and public endpoints. Fix these early, independent of framework migration.
- Retired model versions may auto-upgrade and change behaviour and cost. Evaluate before the dates.
- Archived frameworks (AutoGen) receive no security fixes. Treat them as a vulnerability.

## Common mistakes

| Mistake | Fix |
|---|---|
| Migrating without an evaluation baseline | Baseline first, then compare |
| Porting code mechanically | Re-design around the current agent model |
| Ignoring model retirement dates | Track and test upgrades |
| Starting new work on legacy components | Current stack for all new work |

## Check yourself

1. What replaces the Assistants API?
2. When does prompt flow retire, and what replaces it?
3. What's the status of Semantic Kernel and AutoGen?
4. How do you recognise a hub-based project in IaC?
5. What must you establish before migrating an inherited AI system?

<details>
<summary>Answers</summary>

1. Foundry Agent Service with the Responses API.
2. 20 April 2027; Microsoft Agent Framework and code-first orchestration.
3. Semantic Kernel v1.x gets critical bug and security fixes only, with features moved to Agent Framework. AutoGen is archived.
4. A `Microsoft.MachineLearningServices/workspaces` resource with `kind: 'Hub'`.
5. An evaluation dataset with baseline metrics, plus tracing.

</details>

## References

- [Migrate from Semantic Kernel to Agent Framework](https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/)
- [Migrate from AutoGen to Agent Framework](https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/)
- [Foundry Agent Service overview](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Model deprecations and retirements](https://learn.microsoft.com/azure/foundry/concepts/model-lifecycle-retirement)
- [Microsoft Agent Framework overview](https://learn.microsoft.com/agent-framework/overview/)
