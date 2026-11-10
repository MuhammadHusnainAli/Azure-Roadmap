# Azure Logic Apps

> Topic: 11 · Integration, Messaging and APIs · Tracks: E ○ A ● C ● · Status checked: 2026-11

## Why it matters

Logic Apps is Azure's low-code integration and workflow platform. It has 1,400+ connectors for SaaS, Azure services and on-premises systems, a visual designer, durable run history, and enterprise B2B (EDI) support. It's how many organisations connect ERP, CRM, file drops, email and approvals without writing glue code. Logic Apps can now also run **agentic workflows**: an *agent loop* uses a model to choose which connector-based tools to call. That makes it a practical way to add AI to existing integration estates.

## Key concepts

### Building blocks

| Concept | Meaning |
|---|---|
| **Workflow** | A trigger followed by a series of actions, defined in JSON (Workflow Definition Language) |
| **Trigger** | What starts a run: recurrence, HTTP request, polling a connector, webhook, Service Bus, Event Grid and so on |
| **Action** | A step: call a connector, transform data, condition, loop (For each, Until), scope, call a child workflow |
| **Connector** | A wrapper over an API. **Managed (shared)** connectors run in Microsoft-hosted infrastructure; **built-in** connectors run inside the Logic Apps runtime |
| **Connection** | Stored credentials or identity for a connector |
| **Run history** | Inputs, outputs and status of each step, kept for stateful workflows |

### Consumption vs Standard

| | Consumption | Standard |
|---|---|---|
| Hosting | Multitenant Azure Logic Apps | Single-tenant runtime on a **Workflow Service Plan**, an **App Service Environment v3**, or **hybrid** (your own infrastructure via Container Apps on Arc-enabled Kubernetes) |
| Workflows per resource | One | Many (like functions in a function app) |
| Stateful / stateless | Stateful only | Both: **stateful** (run history in storage, long-running) and **stateless** (in-memory, fast, short) |
| Billing | Pay per execution (actions, connector calls) | Hosting plan (vCPU and memory) plus storage transactions for stateful workflows |
| Connectors | Managed connectors plus some built-ins | Managed connectors plus many **built-in (service provider)** connectors (Service Bus, Event Hubs, SQL, Blob, Cosmos DB, SFTP, MQ...) that run in-process; custom built-in connectors |
| Networking | Public multitenant endpoints; reach private or on-premises resources through the on-premises data gateway | VNet integration, private endpoints, static outbound IPs |
| Local development | Portal-centric | Visual Studio Code with local run and debug; deploy like an app (CI/CD friendly) |
| Data residency | Workflows can replicate to the paired region; agent model calls may be processed in other regions | Data stays in the deployment region |
| Typical use | Light, event-driven automations and prototypes | Enterprise integration, high throughput, private networking, DevOps |

The **Integration Service Environment (ISE)** was retired in August 2024. Standard on a Workflow Service Plan or ASE v3 is the replacement.

### Connectors and identity

- Prefer **managed identity** authentication on connectors that support it (Azure services, HTTP), so you hold no secrets.
- In Standard, the system-assigned identity is enabled by default. Most built-in service-provider connectors support it, but few support selecting a user-assigned identity (SQL Server and HTTP do).
- Reach on-premises systems with the **on-premises data gateway** (managed connectors) or VNet connectivity (Standard).

## How it works on Azure

### B2B and enterprise integration

| Capability | Detail |
|---|---|
| **Integration account** | Stores B2B artefacts: trading partners, agreements, schemas, maps, certificates. Required for EDI in both Consumption and Standard |
| **EDI protocols** | AS2 (transport), X12 and EDIFACT (document standards), RosettaNet |
| **Transforms** | XSLT maps, Liquid templates, flat-file encode/decode. In **Standard**, XML and Liquid operations work **without** an integration account (artefacts live in the logic app) |
| **Tracking** | B2B tracking to Azure Monitor / Log Analytics |
| **Mainframe and midrange** | Connectors for IBM MQ, CICS, IMS, DB2, host files (Standard built-ins) |

### Agentic workflows (agent loops)

Logic Apps can build workflows around an **agent loop** action. The loop sends instructions and inputs to a model, lets the model pick **tools** (each tool is a sequence of one or more Logic Apps actions), runs them, and repeats until the task is done.

| | Autonomous agent workflow | Conversational agent workflow |
|---|---|---|
| Starts with | Any trigger (for example a Service Bus message, email, schedule) | The **When a chat session starts** trigger |
| Human interaction | None; runs to completion | Through an integrated chat interface, or an external chat client secured with Easy Auth (Standard) or Entra OAuth (Consumption) |
| Example | Triage inbound invoices, decide approve or escalate, update ERP | Employee facilities assistant that opens and tracks work orders |

Status and model sources (checked 2026-11):

| | Consumption | Standard |
|---|---|---|
| Status | **Preview** | Generally available (some model sources in preview) |
| Model source | Microsoft Foundry (automatically configured) | Azure OpenAI resource; Microsoft Foundry project (preview); API Management LLM API (preview) |
| Billing | Agent loop tokens billed as Enterprise Units | No extra Logic Apps charge for agents; you pay for model usage |

Key ideas:

- **Agent parameters** are tool inputs the model fills in at call time (for example `city` for a weather tool).
- An agentic workflow can chain multiple agents in sequence, but you **can't** nest an agent as a tool inside another agent.
- Standard logic apps can add an **Agent** action to an existing stateful workflow.
- In the designer, test conversational agents with a portal-managed developer key. For production, use Easy Auth (Standard) or OAuth 2.0 with Entra ID (Consumption).

Logic Apps workflows can also be **tools for other agents**. Foundry Agent Service can call them as tools, and API Management can front a Logic Apps endpoint as an MCP server (file 07). Check current status for "Logic Apps as MCP server" capabilities.

### AI building blocks without an agent

You don't always need an agent loop. Deterministic workflows can call AI as ordinary steps:

```text
When a file is added to SharePoint
  → Document Intelligence / Content Understanding: extract fields
  → Azure OpenAI: summarise and classify (structured output)
  → Condition: amount > 10,000 ?
       yes → Approvals: start approval in Teams
       no  → Dynamics 365: create record
  → Azure AI Search: push document (index for RAG)
```

There are also built-in actions for chunking and parsing documents to support RAG ingestion in Standard (check current action names in the connector reference).

### Deploy a Standard logic app (CLI)

```bash
az logicapp create -g rg-int -n la-orders-prod \
  --storage-account stlaorders --plan asp-ws1 \
  --assign-identity [system]
# Deploy workflows from a VS Code project zip
az logicapp deployment source config-zip -g rg-int -n la-orders-prod --src ./la-orders.zip
```

A Standard project contains `host.json`, `connections.json`, `parameters.json` and one folder per workflow with `workflow.json`. That layout is why it fits normal Git and CI/CD.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Consumption vs Standard | Consumption for low-volume, sporadic workflows and quick wins. Standard for anything enterprise: VNet, many workflows, high throughput, DevOps, data residency |
| Logic Apps vs Functions | Logic Apps when the work is mostly connecting systems and you value the designer, connectors and run history. Functions when the logic is mainly code, compute-heavy or latency-critical |
| Logic Apps vs Power Automate | Power Automate for citizen-developer and personal or team automation in Microsoft 365. Logic Apps for IT-owned, Azure-governed integration with ALM, networking and B2B |
| Agent loop vs deterministic flow | Deterministic when the steps are known and auditable; an agent loop when inputs are unstructured and the path varies. Keep side-effecting tools narrow and add human approval for high-impact actions |
| Logic Apps agent vs Foundry Agent Service | Logic Apps when the agent mainly orchestrates connectors in an integration context; Foundry Agent Service for richer agent features (knowledge, memory, multi-agent, evaluation). They can call each other |
| Stateful vs stateless (Standard) | Stateful for durability, long runs and auditing; stateless for fast, short (under ~5 minutes), small-payload request/response flows |

## Hands-on

1. Create a Consumption logic app: **When an HTTP request is received** → **Compose** → **Response**. Call it with `curl`.
2. Create a Standard logic app in VS Code with a stateful workflow triggered by a Service Bus queue (built-in connector, managed identity). Run it locally, then deploy.
3. Add an **autonomous agent** workflow in Standard. Connect it to an Azure OpenAI deployment, give it instructions to classify support emails, and create two tools: "create ticket" and "send acknowledgement email". Run it with a sample email.
4. Inspect the run history: look at each agent loop iteration, the tool calls the model chose and the token usage.
5. Optional: create an integration account, add two partners and an X12 agreement, and decode a sample 850 purchase order.

## Security, cost and reliability notes

- **Security:** use managed identities on connections and Key Vault references for any secrets. Secure HTTP triggers with Entra ID (Easy Auth or OAuth policies) instead of relying only on SAS callback URLs.
- **Security:** enable **secure inputs/outputs** on steps that handle sensitive data so it doesn't appear in run history.
- **Security:** agent tools act with the workflow's connections. Apply least privilege and put approvals in front of high-impact actions.
- **Cost:** Consumption is cheap for low volume but grows with every action, so a polling trigger that fires every 10 seconds costs money even when idle. Standard is a fixed plan cost, which is better for steady volume.
- **Cost:** agent loops consume model tokens on every iteration; limit iterations and keep instructions concise.
- **Reliability:** configure retry policies per action, and use **scopes** with run-after conditions (try/catch/finally) for error handling.
- **Reliability:** for Standard, the storage account is critical: protect it and monitor it.

## Common mistakes

- Choosing Consumption for a high-volume or VNet-bound integration, then rebuilding on Standard.
- Leaving sensitive payloads visible in run history.
- Polling triggers at aggressive intervals on Consumption.
- Long chains of nested conditions instead of child workflows or scopes.
- Treating agent loops as deterministic and skipping evaluation, guardrails and approvals.
- Forgetting that stateless workflows don't keep run history unless you enable it for debugging.

## Check yourself

1. Name four differences between Consumption and Standard logic apps.
2. What replaced the Integration Service Environment?
3. What is the difference between built-in and managed connectors?
4. What is an agent loop, and what is a tool in Logic Apps?
5. How do autonomous and conversational agentic workflows differ?
6. When do you need an integration account in Standard?
7. When would you choose Functions instead of Logic Apps?

<details>
<summary>Answers</summary>

1. Multitenant vs single-tenant hosting; one vs many workflows per resource; stateful only vs stateful and stateless; pay per execution vs hosting plan; limited vs full VNet integration and private endpoints; portal-centric vs local VS Code development.
2. Standard logic apps on a Workflow Service Plan or App Service Environment v3.
3. Built-in connectors run inside the Logic Apps runtime (in-process, faster, packaged with the app). Managed connectors run in Microsoft-hosted shared infrastructure and are created as separate API connection resources.
4. An agent loop is a prebuilt action that iteratively uses a model to reason, choose tools and act. A tool is a sequence of one or more Logic Apps actions the agent can invoke, with agent parameters the model fills in.
5. Autonomous workflows start from any trigger and run without human interaction. Conversational workflows start when a chat session starts and interact with a person through a chat interface.
6. For B2B/EDI scenarios (trading partners, agreements, AS2/X12/EDIFACT). XML and Liquid transforms in Standard don't need one.
7. When the logic is code-heavy, compute-intensive, latency-sensitive, or doesn't benefit from connectors and the designer.

</details>

## References

- [What is Azure Logic Apps?](https://learn.microsoft.com/azure/logic-apps/logic-apps-overview)
- [Standard vs Consumption logic apps](https://learn.microsoft.com/azure/logic-apps/single-tenant-overview-compare)
- [AI agentic workflows in Logic Apps](https://learn.microsoft.com/azure/logic-apps/agent-workflows-concepts)
- [Create autonomous agentic workflows](https://learn.microsoft.com/azure/logic-apps/create-autonomous-agent-workflows)
- [B2B enterprise integration overview](https://learn.microsoft.com/azure/logic-apps/logic-apps-enterprise-integration-overview)
- [Built-in connectors](https://learn.microsoft.com/azure/logic-apps/connectors/built-in)
- [Logic Apps pricing model](https://learn.microsoft.com/azure/logic-apps/logic-apps-pricing)
- [Secure access and data](https://learn.microsoft.com/azure/logic-apps/logic-apps-securing-a-logic-app)
