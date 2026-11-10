# Foundry Toolkit for VS Code and Developer Workflow

> Topic: 14 · Microsoft Foundry Platform · Tracks: E ● A – C – · Status checked: 2026-11

## Why it matters

Building AI apps means iterating fast on prompts, models, tools and agents, then moving the result reliably into a governed environment. Microsoft's developer loop for Foundry has three parts. The **Foundry Toolkit for Visual Studio Code** handles build, test, evaluate and trace in the editor. The **Azure Developer CLI (`azd`)** templates and AI agent extension handle provisioning and deployment. The **Foundry portal playgrounds** handle quick experiments. Engineers who use this loop well spend less time on plumbing and ship with evaluations and traces from day one.

## Key concepts

### Foundry Toolkit for Visual Studio Code

The **Foundry Toolkit** is the new name for the **AI Toolkit** extension, and it absorbs the capabilities of the separate Microsoft Foundry extension, so local and Foundry resources are managed in one place. It reached general availability in April 2026, with some experiences still in preview.

| Area | Features (checked 2026-11) |
|---|---|
| **Build** | **Agent Builder** (prompt agents: instructions, tools, connections, versions); hosted-agent samples and Copilot-assisted coding; **Tool Catalog** (external tools, knowledge resources); **Workflows**; **Routines** (Preview: schedule or trigger an agent) |
| **Explore models** | **Model Catalog** (deploy Foundry models, connect provider endpoints, add local models); **Model Playground** (prompts, parameters, side-by-side comparison, image generation, code samples) |
| **Customise** | Fine-tuning, model conversion, **Model Profiling** (Preview) |
| **Test and evaluate** | **Agent Inspector** (call a locally running agent, inspect responses and tool calls); **Evaluation** with datasets, built-in or custom evaluators, locally or in Foundry |
| **Monitor** | **Tracing** of local telemetry or hosted-agent traces |
| **Deploy** | **Hosted-agent deployment** (source code or container image) and the Hosted Agent Playground; manage models, agents, tools, knowledge and evaluations in a selected project |

The sidebar has three sections: **My Resources** (Foundry project and local resources), **Developer Tools** (Discover, Build, Monitor) and **Help and Feedback**. Local models run through **Foundry Local** (file 08).

### Azure Developer CLI (`azd`)

| Concept | Use |
|---|---|
| **Templates** | Repos with app code + Bicep/Terraform + `azure.yaml`; `azd init -t <template>` then `azd up` provisions and deploys |
| **AI agent extension** (`azd ai agent`) | Initialises, deploys and publishes Foundry agents (including hosted agents) from code; installed automatically when needed |
| **Environments** | `azd env new dev` / `prod` with separate parameters |
| **CI/CD** | `azd pipeline config` sets up GitHub Actions or Azure Pipelines with federated credentials |

Official AI templates are listed in the Foundry portal's template catalogue and in the Azure Developer CLI template gallery (search "AI"). Template names change, so pick from the gallery rather than from old blog posts.

### Portal playgrounds

| Playground | Use |
|---|---|
| Chat / model playground | Prompt iteration, parameters, compare models, view code |
| Agents playground | Build and test prompt agents with tools and knowledge |
| Images, audio, video playgrounds | Try generation and speech models |
| Evaluation | Run evaluations on datasets, compare runs |

Playgrounds are for exploration. Anything that matters should end up in code and source control.

## How it works on Azure

### The inner loop

```text
1. Explore      Foundry portal / Toolkit Model Catalog + Playground   → pick candidate models
2. Prototype    Toolkit Agent Builder or code (Foundry SDK / Agent Framework) locally
3. Inspect      Agent Inspector + local tracing (OpenTelemetry)          → fix tool calls, prompts
4. Evaluate     Toolkit Evaluation with a small dataset (local or in Foundry)
5. Commit       Prompts, agent definitions, eval datasets, IaC → Git
```

### The outer loop

```text
6. Provision    azd up (dev env) → Foundry resource, project, deployments, connections, App Insights
7. Deploy       azd deploy / azd ai agent deploy → agent versions, app (Container Apps / App Service)
8. CI           PR pipeline: lint, unit tests, evaluation gate (quality and safety thresholds)
9. CD           Promote to test/prod environments (azd env, or Bicep/Terraform in pipelines)
10. Operate     Foundry Control Plane: tracing, continuous evaluation, alerts, red teaming
```

### Start from a template

```bash
# Official starter used by the azd AI agent extension docs (check the gallery for current templates)
azd init -t Azure-Samples/azd-ai-starter-basic
azd env new dev
azd up                 # provisions Foundry resource, project, model deployment, and deploys the sample agent
azd ai agent --help    # explore agent-specific commands (init, deploy, invoke...)
```

Some features (for example hosted agents) are available only in specific regions; the template README states them.

### Local tracing and evaluation in code (sketch)

```python
# Enable OpenTelemetry tracing to Application Insights connected to the project
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.monitor.opentelemetry import configure_azure_monitor

project = AIProjectClient(endpoint="https://<res>.services.ai.azure.com/api/projects/<proj>",
                          credential=DefaultAzureCredential())
conn_str = project.telemetry.get_application_insights_connection_string()
configure_azure_monitor(connection_string=conn_str)
# ...then instrument the OpenAI client / agent calls; traces appear in Foundry and App Insights
```

Method names for telemetry differ by SDK version; topic 22 covers tracing and evaluation in depth.

### GitHub Copilot and MCP in the loop

- The Toolkit's Copilot-assisted coding and Foundry-aware prompts help scaffold agents and hosted agents.
- The **Azure MCP Server** and **Foundry MCP Server** let GitHub Copilot agent mode inspect and operate Azure and Foundry resources from chat (topic 04 file 10).
- Treat AI-generated IaC and code like any other contribution: review and test it.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Portal vs Toolkit vs code | Portal for discovery and demos; Toolkit for daily development in VS Code; code + IaC as the source of truth |
| azd vs bespoke pipelines | azd for fast start and consistent environments; bespoke Bicep/Terraform pipelines for complex enterprise landing zones (azd can still wrap them) |
| Prompt agents vs hosted agents | Prompt agents (configured, no code hosting) for most cases; hosted agents (your code in a container, Agent Framework or other frameworks) for custom logic (topic 18) |
| Local models in the loop | Use Foundry Local for offline experiments, cheap iteration and edge targets; always evaluate on the production model before release |
| Evaluation gates | Add a small but meaningful eval dataset to CI from the first sprint |

## Hands-on

1. Install the **Foundry Toolkit** extension, sign in, and select your Foundry project under **My Resources**.
2. In the Model Playground, compare two deployed models on five prompts and copy the generated code sample.
3. Build a prompt agent in **Agent Builder** with one tool, run it in the Agent Inspector, and inspect the tool call.
4. Run an **Evaluation** on a 20-row dataset with built-in evaluators.
5. Use `azd init -t Azure-Samples/azd-ai-starter-basic` and `azd up` to provision a full environment; then `azd down --purge` to clean up.
6. Run `azd pipeline config` to create a GitHub Actions workflow with federated credentials.

## Security, cost and reliability notes

- **Security:** azd and the Toolkit use your signed-in identity locally. Pipelines should use workload identity federation (no secrets) with least-privilege roles.
- **Security:** don't commit `.env` files with keys. Use Entra ID and Key Vault references.
- **Cost:** `azd up` creates real resources, some with standing costs (AI Search, Cosmos DB, Container Apps minimum replicas). Use `azd down --purge` after experiments; Foundry resources are soft-deleted and must be purged to free names and quota.
- **Reliability:** pin template versions or fork them. Upstream templates change.

## Common mistakes

- Keeping prompt and agent changes only in the portal, so they're not reproducible.
- Using outdated AI Toolkit tutorials without noticing the rename and new features.
- Forgetting to purge soft-deleted resources, then hitting name conflicts or quota limits.
- No evaluation step before deployment.
- Running templates in regions that don't support the features they need.

## Check yourself

1. What is the Foundry Toolkit, and what did it replace?
2. Name four Foundry Toolkit capabilities.
3. What does `azd up` do?
4. What is the `azd ai agent` extension for?
5. Describe the inner and outer development loops for a Foundry agent.
6. Why must you purge deleted Foundry resources?

<details>
<summary>Answers</summary>

1. A VS Code extension for building, testing, evaluating, tracing and deploying AI apps and agents with Foundry and local models; it's the renamed AI Toolkit and includes the former Microsoft Foundry extension's capabilities.
2. Any four of: Agent Builder, Tool Catalog, Model Catalog, Model Playground, Agent Inspector, Evaluation, Tracing, fine-tuning, hosted-agent deployment, workflows.
3. Provisions the template's infrastructure and deploys its application code in one step.
4. Initialising, deploying and publishing Foundry agents (including hosted agents) from code.
5. Inner: explore models, prototype, inspect and trace locally, evaluate, commit. Outer: provision with IaC, deploy, run CI with evaluation gates, promote through environments, operate with monitoring and continuous evaluation.
6. Deleted Foundry (Cognitive Services) resources are soft-deleted and keep their name (and can affect quota) until purged.

</details>

## References

- [Foundry Toolkit for Visual Studio Code](https://code.visualstudio.com/docs/intelligentapps/overview)
- [Get started with Foundry projects in VS Code](https://learn.microsoft.com/azure/foundry/how-to/develop/get-started-projects-vs-code)
- [Azure Developer CLI AI agent extension](https://learn.microsoft.com/azure/developer/azure-developer-cli/extensions/azure-ai-foundry-extension)
- [Azure Developer CLI overview](https://learn.microsoft.com/azure/developer/azure-developer-cli/overview)
- [Prepare your development environment for Foundry](https://learn.microsoft.com/azure/foundry/how-to/develop/install-cli-sdk)
- [Recover or purge deleted Foundry resources](https://learn.microsoft.com/azure/ai-services/recover-purge-resources)
