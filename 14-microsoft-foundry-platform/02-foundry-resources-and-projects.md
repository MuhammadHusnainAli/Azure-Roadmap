# Foundry Resources and Projects

## Why it matters

How you lay out Foundry resources and projects decides who can see what, where data lives, how costs are attributed, how networking works and how teams scale. A bad layout (one resource for the whole company, or a resource per developer) is painful to undo. You also need to know the older **hub-based** model, because many organisations still run it, and how to migrate from it.

## Key concepts

### Foundry resource

The **Foundry resource** (`Microsoft.CognitiveServices/accounts`, kind `AIServices`) is the top-level Azure resource and the **governance boundary**:

| Managed at resource level | Notes |
|---|---|
| **Model deployments** | Shared by all projects in the resource |
| **Networking** | Public network access, private endpoints, network injection for agents |
| **Encryption** | Microsoft-managed or customer-managed keys |
| **Resource-level connections** | Shared connections (for example a Key Vault connection, shared AI Search) |
| **Guardrails / content filters** | Per deployment, plus guardrail policies |
| **Local auth** | Enable or disable API keys (`disableLocalAuth`) |
| **Monitoring** | Resource-level metrics (tokens, latency, errors), diagnostic settings |
| **AI gateway association** | Link to API Management (file 09) |

`allowProjectManagement: true` must be set when the resource is created to allow projects. It can't be changed afterwards.

### Foundry project

A **project** (`Microsoft.CognitiveServices/accounts/projects`) is a **development boundary** inside the resource:

| Scoped to the project | Notes |
|---|---|
| Agents, agent versions, conversations | Agent Service artefacts |
| Files, datasets, vector stores, indexes | Project assets |
| Evaluations and evaluation results | |
| Project connections | Connections only that project can use |
| Project **managed identity** | Used by agents and tools to reach other resources |
| RBAC at project scope | Developers can be limited to their project |
| Project-level metrics and tracing | With Application Insights connected |

A resource can contain many projects. Each new resource gets a **default project**, and some features at the resource level (for example some Foundry Tools APIs such as Translator) are available only through the resource endpoint, not the project endpoint.

### Standalone Azure OpenAI resources

A resource of kind `OpenAI` is still supported and exposes only the OpenAI endpoints. Microsoft provides an **upgrade path from Azure OpenAI to Foundry**: the resource is upgraded in place to kind `AIServices`, keeping its endpoint, deployments, keys, RBAC and policies, and gaining projects (check prerequisites and limitations before upgrading).

### The older hub-based model (Foundry classic)

| | Hub-based | Foundry resource + projects |
|---|---|---|
| Top-level resource | **Hub** (`Microsoft.MachineLearningServices/workspaces`, kind `Hub`) | **Foundry resource** (`Microsoft.CognitiveServices/accounts`, kind `AIServices`) |
| Project resource | Hub-based project (`MachineLearningServices/workspaces`, kind `Project`) | `CognitiveServices/accounts/projects` |
| Model access | Through a connected Azure OpenAI / AI services resource | Native deployments on the resource |
| Dependent resources | Storage account, Key Vault, optional ACR and App Insights created with the hub | Microsoft-managed by default; bring your own via connections |
| Networking | Azure ML managed VNet | Private endpoints + agent network injection or managed VNet |
| Roles | Azure ML roles (Azure AI Developer, AzureML Data Scientist) | Foundry roles (Foundry User, Owner...) |
| Compute | Compute instances, serverless compute, prompt flow runtimes | Fully managed; no compute instances |
| SDK | `azure-ai-projects` 1.x / azure-ai-ml | `azure-ai-projects` 2.x |

Use hub-based projects only when you need a feature that isn't yet available in Foundry projects (check the feature comparison table; prompt flow and some Azure ML-integrated features have historically been hub-only).

### Migrating from hub-based to Foundry projects

Microsoft's guidance (checked 2026-11) is to **create new Foundry projects** rather than convert hubs in place:

1. Create (or reuse) a Foundry resource, possibly by upgrading the Azure OpenAI resource the hub was connected to, so deployments carry over.
2. Create Foundry projects for each hub-based project.
3. Recreate **connections** (storage, search, external APIs) on the Foundry resource or project.
4. Migrate agents (code-based agents may need updates to the new Agent Service APIs and SDK 2.x), evaluations and datasets.
5. Update application endpoints and roles.
6. Delete hub-based projects when no longer needed.

There's no full feature parity yet, so check the support matrix before you migrate.

## How it works on Azure

### Create with the CLI

```bash
RG=rg-foundry-dev; LOC=swedencentral; NAME=fdy-contoso-dev

az cognitiveservices account create -g $RG -n $NAME -l $LOC \
  --kind AIServices --sku S0 \
  --custom-domain $NAME \
  --allow-project-management \
  --assign-identity

az cognitiveservices account project create -g $RG --name $NAME \
  --project-name claims-assistant --location $LOC

az cognitiveservices account deployment create -g $RG -n $NAME \
  --deployment-name gpt-5-mini --model-name gpt-5-mini --model-version "2025-08-07" \
  --model-format OpenAI --sku-name GlobalStandard --sku-capacity 50
```

The custom subdomain is required for Entra ID authentication. Check `az cognitiveservices account project --help` for current parameters, since these commands are relatively new.

### Layout patterns

| Pattern | Description | Use when |
|---|---|---|
| **Resource per environment, projects per team or use case** | `fdy-<org>-dev`, `-test`, `-prod`; projects for each team or app | Most organisations; clear environment isolation |
| **Resource per business unit** | Each BU owns a resource with its own networking, quotas and budgets | Strong chargeback or isolation needs |
| **Resource per application (production)** | Dedicated resource per critical app | Independent lifecycle, networking, keys, quotas |
| **Shared "AI hub" resource + AI gateway** | Central platform team owns deployments; apps consume via APIM | Central capacity management (PTU), consistent governance (see topic 03 file 04) |

Remember that deployments are **resource-level**: projects in the same resource share deployments and their TPM quota. If teams need separate quota or capacity guarantees, give them separate deployments or separate resources.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Granularity | Resources for governance boundaries (network, keys, region, environment); projects for team and use-case isolation |
| Region | Choose by model availability, data residency and latency; you may need resources in more than one region for some models |
| Shared vs dedicated deployments | Shared for cost efficiency in dev/test; dedicated (per app) in production when you need guaranteed capacity or independent version upgrades |
| Upgrade Azure OpenAI vs new resource | Upgrade in place to keep endpoints, deployments and RBAC; create new when you need a fresh network or encryption design |
| Hub migration timing | Migrate when the features you use are available in Foundry projects; new work goes straight to Foundry projects |

## Hands-on

1. Create a Foundry resource with `--allow-project-management` and two projects (`team-a`, `team-b`).
2. Deploy one model and confirm it's visible from both projects' playgrounds.
3. Give a colleague **Foundry User** on `team-a` only and confirm they can't see `team-b`'s agents.
4. In the Azure portal, inspect the resource JSON and find `allowProjectManagement`, `customSubDomainName` and the projects child resources.
5. If you have a hub-based project, list its connections and write a migration checklist.

## Security, cost and reliability notes

- **Security:** project isolation covers project assets and RBAC, but deployments and resource-level connections are shared. Don't put workloads with different data-classification requirements in the same resource without reviewing this.
- **Security:** each project has a managed identity. Grant it only the roles its agents need on connected resources.
- **Cost:** tag resources (and use project-level metrics) for chargeback. Separate resources give the cleanest billing separation (file 10).
- **Reliability:** a resource lives in one region. Plan multi-region deployments explicitly.

## Common mistakes

- Creating the resource without `allowProjectManagement`, then being unable to add projects.
- One resource for dev and prod, sharing quota and deployments.
- Expecting project-level isolation for model deployments.
- Starting new work on hubs.
- Migrating hubs without checking feature parity.

## Check yourself

1. What is managed at the Foundry resource level vs the project level?
2. Which property must be set at creation to allow projects?
3. How does a hub-based project differ from a Foundry project in resource types?
4. What's Microsoft's recommended approach for migrating hub-based projects?
5. Why might two teams need separate model deployments even in the same resource?
6. What happens when you upgrade an Azure OpenAI resource to Foundry?

<details>
<summary>Answers</summary>

1. Resource: model deployments, networking, encryption, shared connections, guardrails, local auth, monitoring. Project: agents, files, datasets, evaluations, project connections, project managed identity, project-scoped RBAC.
2. `allowProjectManagement` (`--allow-project-management` in the CLI).
3. Hubs and hub-based projects are `Microsoft.MachineLearningServices/workspaces` (kinds `Hub` and `Project`); Foundry projects are `Microsoft.CognitiveServices/accounts/projects` under a Foundry resource.
4. Create new Foundry projects (on a Foundry resource, possibly an upgraded Azure OpenAI resource), recreate connections, migrate agents and assets, update apps, then delete the hub-based projects.
5. Deployments carry TPM quota and version settings; separate deployments give independent capacity, rate limits and upgrade schedules.
6. It becomes kind `AIServices` in place, keeping endpoint, deployments, keys and RBAC, and gains project, agent and evaluation capabilities.

</details>

## References

- [Microsoft Foundry architecture](https://learn.microsoft.com/azure/foundry/concepts/architecture)
- [Create a project](https://learn.microsoft.com/azure/foundry/how-to/create-projects)
- [Upgrade from Azure OpenAI to Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/upgrade-azure-openai)
- [Migrate from hub-based to Foundry projects (classic)](https://learn.microsoft.com/azure/foundry-classic/how-to/migrate-project)
- [Hub-based projects (classic)](https://learn.microsoft.com/azure/foundry-classic/concepts/ai-resources)
- [Foundry rollout planning](https://learn.microsoft.com/azure/foundry/concepts/planning)
