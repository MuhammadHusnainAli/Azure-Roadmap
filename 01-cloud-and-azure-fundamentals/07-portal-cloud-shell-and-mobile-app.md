# Portal, Cloud Shell and Mobile App

> Topic: 01 · Cloud and Azure Fundamentals · Tracks: E ● A ○ C ○ · Status checked: 2026-10

## Why it matters

The portal is where you explore, troubleshoot and demonstrate. It is not where production gets built, but an engineer who cannot find an activity log entry, a consultant who fumbles a demo, or an architect who cannot query the estate all lose time and credibility. This file covers the interactive tools; topic 04 covers the automation tools.

## Key concepts

### The management interfaces

| Interface | Best for | Not for |
|---|---|---|
| **Azure portal** (`portal.azure.com`) | Exploring, one-off tasks, troubleshooting, dashboards | Repeatable deployments |
| **Azure Cloud Shell** | Quick CLI or PowerShell from any browser, no install | Long-running jobs, heavy development |
| **Azure mobile app** | Alerts, health checks, quick restarts on the move | Building anything |
| **Azure CLI / PowerShell** | Scripting and automation | (topic 04) |
| **Bicep / Terraform** | Repeatable infrastructure | (topic 04) |
| **Microsoft Foundry portal** (`ai.azure.com`) | Building and testing AI: models, agents, evaluations | General Azure resource management |

All of them are clients of Azure Resource Manager. Nothing the portal can do is unavailable to the API.

The Foundry portal is a separate site focused on AI development. You create and secure the Foundry *resource* through Azure (portal or IaC), and you work with models, agents and evaluations in the Foundry portal. Expect to have both open.

### Portal anatomy

| Element | What it does |
|---|---|
| **Global search** (`G` then `/`) | Finds resources, services, documentation and Marketplace items. The fastest way to navigate |
| **Portal menu** | Favourites and *All services* |
| **Directory + subscription filter** (settings gear) | Chooses which tenant you are in and which subscriptions are shown |
| **Cloud Shell icon** | Opens a terminal in the browser |
| **Copilot icon** | Opens Azure Copilot |
| **Notifications** (bell) | Progress and result of operations you started |
| **Breadcrumbs** | Where you are in the current navigation path |

Each resource page has the same left-menu structure, which makes unfamiliar services navigable:

| Section | Contents |
|---|---|
| **Overview** | Essentials (resource group, location, subscription, ID), key actions, **JSON View** |
| **Activity log** | Control-plane operations on this resource |
| **Access control (IAM)** | Role assignments, *Check access* |
| **Tags** | Tag editor |
| **Diagnose and solve problems** | Guided troubleshooting |
| **Settings** | Service-specific configuration, plus Locks and Properties |
| **Monitoring** | Metrics, alerts, diagnostic settings, logs |
| **Automation** | **Export template**, tasks |
| **Help** | Resource health, support request |

Two of these earn their keep daily:

- **JSON View** shows the resource exactly as ARM sees it, with an API version selector. Use it to find the property name you need for Bicep or Terraform.
- **Export template** generates ARM JSON or Bicep for an existing resource. The output is verbose and needs cleaning, but it is a fast way to learn a resource's shape.

### The "wrong directory" problem

If a resource or subscription seems to be missing, you are almost always in the wrong tenant or have a subscription filter applied. Open **Settings → Directories + subscriptions**, switch directory if needed, and set the default subscription filter to include what you expect. Consultants working across client tenants hit this daily.

### Dashboards and workbooks

- **Dashboards** are customisable tile layouts. They can be private or shared (a shared dashboard is itself an Azure resource with RBAC). Good for an at-a-glance operations view.
- **Azure Monitor workbooks** are interactive reports combining text, queries, metrics and parameters. Better for analysis and investigation.
- **Dashboards with Grafana** in the portal and **Azure Managed Grafana** cover teams standardised on Grafana.

Covered in topic 26.

### Azure Resource Graph Explorer

Resource Graph lets you query every resource you can see, across subscriptions, using KQL. It is fast, read-only, and does not consume the ARM read throttling budget in the same way as looping GET calls.

```kusto
// Count resources by type
Resources
| summarize count() by type
| order by count_ desc

// Resources missing an 'owner' tag
Resources
| where isnull(tags.owner)
| project name, type, resourceGroup, subscriptionId

// Storage accounts allowing public blob access
Resources
| where type =~ 'microsoft.storage/storageaccounts'
| where properties.allowBlobPublicAccess == true
| project name, resourceGroup, location

// All Foundry / AI services resources and their kinds
Resources
| where type =~ 'microsoft.cognitiveservices/accounts'
| project name, kind, location, sku = sku.name, resourceGroup

// Public IP addresses and what they are attached to
Resources
| where type =~ 'microsoft.network/publicipaddresses'
| project name, ip = properties.ipAddress, attachedTo = properties.ipConfiguration.id
```

For an architect or consultant doing a current-state assessment, Resource Graph is the single most useful portal feature. Topic 03 goes deeper.

### Azure Copilot

Azure Copilot is the AI assistant built into the portal (and available in the mobile app and CLI). It answers questions about your environment, generates CLI, PowerShell, KQL and IaC, explains errors and costs, and, through its agent capabilities, can carry out multi-step operational tasks such as troubleshooting, migration planning and optimisation with your approval.

Things to understand about it:

- It acts **with your permissions**. It cannot see or change what your identity cannot.
- Tenant administrators can **control who has access** to it.
- Treat generated commands and queries as drafts. Read them before running.
- It is good at "what is this error", "write the KQL for", and "which resources are affected by this service issue".

Covered in topic 24.

### Cloud Shell

A browser-based shell, authenticated as you, running on a temporary Linux container.

- **Shells:** Bash or PowerShell, switchable.
- **Pre-installed:** Azure CLI, Azure PowerShell, Bicep, Terraform, `azd`, kubectl, Helm, git, Python, Node.js, .NET, a built-in editor (`code .`), and more.
- **Access:** portal icon, `shell.azure.com`, the mobile app, VS Code.

**Storage modes:**

| Mode | Behaviour |
|---|---|
| **Ephemeral** | No storage account. Fast to start. Files vanish when the session ends |
| **Mounted storage** | An Azure Files share is mounted so `$HOME` persists across sessions. You pay for the share |

**Limitations:**

- Sessions time out after about 20 minutes of inactivity.
- The container is temporary. Only `$HOME` persists, and only with mounted storage.
- By default it runs on Microsoft's network and reaches your resources over public endpoints, so it **cannot reach private endpoints**. Deploying Cloud Shell **into your virtual network** fixes this at the cost of extra resources.
- Not suitable for long deployments or background jobs.
- It authenticates as the signed-in user automatically. Some operations (for example certain Microsoft Graph calls) may need a separate sign-in.

Many organisations disable Cloud Shell or restrict it, because it is an outbound-capable shell that bypasses device controls. Don't assume a client allows it.

### Azure mobile app

Available for iOS and Android. Useful for:

- Receiving and acknowledging alerts and Service Health notifications.
- Checking resource health and metrics.
- Start, stop, restart of VMs and web apps.
- Running Cloud Shell and using Azure Copilot.
- Approving Privileged Identity Management activation requests.

It is an on-call tool.

### Other portals you will meet

| Portal | URL | Purpose |
|---|---|---|
| Microsoft Foundry | `ai.azure.com` | AI models, agents, evaluations |
| Microsoft Entra admin centre | `entra.microsoft.com` | Identity |
| Microsoft Defender | `security.microsoft.com` | Security operations, Sentinel |
| Microsoft Purview | `purview.microsoft.com` | Data governance and compliance |
| Microsoft Fabric | `app.fabric.microsoft.com` | Analytics |
| Copilot Studio | `copilotstudio.microsoft.com` | Low-code agents |
| Microsoft 365 admin centre | `admin.microsoft.com` | Licences, Microsoft 365 services |
| Azure DevOps | `dev.azure.com` | Repos, pipelines, boards |
| Azure status | `azure.status.microsoft` | Public platform status |
| Preview portal | `preview.portal.azure.com` | Upcoming portal features |

Knowing which portal owns a task is part of being fluent in the Microsoft ecosystem.

## How it works on Azure

### A portal-to-code workflow that works

1. Create the resource once in the portal in a sandbox to learn its options.
2. On the last page of the create wizard, select **Download a template for automation** before clicking Create, or use **Export template** afterwards.
3. Open **JSON View** to confirm property names and the API version.
4. Rewrite it cleanly in Bicep or Terraform (or decompile ARM JSON with `az bicep decompile`).
5. Delete the hand-made resource and deploy from code.

### Useful portal settings

- **Settings → Appearance + startup views:** theme, default landing view.
- **Settings → Language + region.**
- **Settings → Signing out + notifications:** inactivity sign-out. Directory-level timeout can be enforced by an admin.
- **Favourites:** pin the dozen services you actually use.
- **Keyboard shortcuts:** `G+/` search, `G+D` dashboard, `G+A` all resources, `G+R` resource groups, `?`-style help via the keyboard shortcuts menu.

## Design decisions and trade-offs

- **Portal changes cause drift.** If a resource is managed by IaC, a portal edit will be silently reverted or will make the next deployment fail. In production, give most people Reader in the portal and make changes through pipelines.
- **Cloud Shell vs local tooling.** Cloud Shell is perfect for a locked-down laptop or a quick fix. For daily work, local tools in a dev container or Dev Box are faster and version-pinned.
- **Shared dashboards are resources.** Put them in a resource group, define them as code, and grant access with RBAC.
- **Copilot is an accelerator, not an authority.** Use it to draft; verify before you apply.

## Hands-on

1. **Customise:** pin Resource groups, Subscriptions, Cost Management + Billing, Monitor, Microsoft Entra ID, Resource Graph Explorer and Policy to favourites.
2. **Directory filter:** open *Directories + subscriptions* and note which tenant and subscriptions are active.
3. **Cloud Shell:** start it in ephemeral mode and run:

   ```bash
   az account show -o table
   az version
   bicep --version
   terraform version
   az group list --query "[].{Name:name, Location:location}" -o table
   ```

4. **Resource Graph:** open Resource Graph Explorer and run the first three queries above. Pin one result to a new dashboard.
5. **JSON View:** open any resource, select *JSON View*, change the API version and watch the properties change.
6. **Export:** on a resource group, open *Export template*, switch to the Bicep tab and read the output.
7. **Copilot:** ask it "Which of my resources have public network access enabled?" and review the query it generates.
8. **Activity log:** find the entry for something you did in an earlier lab and open its JSON to see the caller, IP address and authorisation details.

## Security, cost and reliability notes

- **Security:** require MFA and compliant devices for portal access with Conditional Access targeting *Windows Azure Service Management API* / Azure management. Portal access is control-plane access.
- **Security:** Cloud Shell runs as you. Anything you paste runs with your permissions. Be careful with scripts from the internet.
- **Cost:** mounted Cloud Shell storage is a small ongoing charge. Ephemeral mode is free.
- **Reliability:** the portal can be degraded while the platform is healthy. If the portal misbehaves during an incident, the CLI often still works. Check `azure.status.microsoft` and Service Health.

## Common mistakes

- Hunting for a "missing" resource while in the wrong directory or with a subscription filter on.
- Building production by clicking and having no record of what was configured.
- Expecting Cloud Shell to reach private endpoints.
- Leaving a long deployment running in Cloud Shell and losing it to the idle timeout.
- Running Copilot-generated commands without reading them.
- Looking for agents and model playgrounds in the Azure portal instead of the Foundry portal.
- Sharing a screenshot of the portal that shows subscription IDs, keys or connection strings.

## Check yourself

1. Why can every portal action also be done through the CLI?
2. A colleague cannot see a subscription you know exists. What are the two most likely causes?
3. What is JSON View useful for?
4. What are the two Cloud Shell storage modes and the trade-off?
5. Why can't default Cloud Shell reach a storage account that only has a private endpoint?
6. What is Azure Resource Graph, and why prefer it to looping over `az resource list`?
7. With whose permissions does Azure Copilot act?
8. Which portal do you use to build an agent, and which to configure its network isolation?

<details>
<summary>Answers</summary>

1. The portal is a client of Azure Resource Manager, the same API the CLI calls.
2. They are in a different directory, or their subscription filter excludes it. (Third: they have no role on it.)
3. Seeing the exact ARM representation and property names of a resource for a given API version, for use in IaC and troubleshooting.
4. Ephemeral (fast, free, nothing persists) and mounted storage (persists `$HOME` on an Azure Files share that you pay for).
5. It runs on Microsoft's network outside your VNet, so it uses public endpoints. VNet-deployed Cloud Shell solves this.
6. A service that indexes resource metadata for fast KQL queries across subscriptions. It is quicker, supports rich filtering and aggregation, and avoids ARM throttling from many GET calls.
7. The signed-in user's.
8. The Foundry portal to build the agent; the Azure portal or IaC to configure the Foundry resource's networking.

</details>

## References

- [What is the Azure portal?](https://learn.microsoft.com/azure/azure-portal/azure-portal-overview)
- [Manage Azure portal settings and preferences](https://learn.microsoft.com/azure/azure-portal/set-preferences)
- [What is Azure Cloud Shell?](https://learn.microsoft.com/azure/cloud-shell/overview)
- [Deploy Cloud Shell in a virtual network](https://learn.microsoft.com/azure/cloud-shell/vnet/overview)
- [What is Azure Resource Graph?](https://learn.microsoft.com/azure/governance/resource-graph/overview)
- [Starter Resource Graph query samples](https://learn.microsoft.com/azure/governance/resource-graph/samples/starter)
- [Azure Copilot overview](https://learn.microsoft.com/azure/copilot/overview)
- [Azure mobile app](https://learn.microsoft.com/azure/azure-portal/mobile-app/overview)
