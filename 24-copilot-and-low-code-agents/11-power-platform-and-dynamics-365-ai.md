# Power Platform and Dynamics 365 AI

## Why it matters

A large share of business processes already run on Power Apps, Power Automate, Dataverse and Dynamics 365 (Sales, Customer Service, Field Service, Finance, Supply Chain). AI there arrives in three forms: **Copilot** features built into the apps (summaries, drafting, natural-language app and flow creation), **AI Builder** models and prompts used inside apps and flows, and **agents** (Microsoft-provided Dynamics 365 agents and agents built in Copilot Studio over Dataverse). Many of the best AI use cases are "inside" these business applications, close to the data and process. Architects need to know where Power Platform AI meets Azure: Foundry models in prompts, Azure AI Search and Azure APIs as connectors, private networking via VNet support, data flowing to Fabric, and governance that spans both worlds.

## Key concepts

### AI capabilities by layer

| Layer | Capability | Example |
|---|---|---|
| **Copilot in makers' tools** | Build apps, flows, tables and pages from natural language | "Create an app to track equipment inspections" |
| **Copilot in apps (end users)** | Model-driven app Copilot, Dynamics 365 Copilot features | Lead summary, email drafting, case summarisation, knowledge article drafting |
| **AI Builder** | Prebuilt models (document processing, receipts, invoices, text classification, sentiment) and custom models trained on your data | Extract invoice fields in a Power Automate flow |
| **Prompts** (AI Builder/Copilot Studio prompt tool) | Reusable GPT prompts with inputs and Dataverse grounding; managed models or **Foundry models** | Classify a support email and return JSON |
| **Agents** | Dynamics 365 agents (for example sales, customer service, supply chain agents) and Copilot Studio agents over Dataverse | Agent qualifies inbound leads and drafts outreach |
| **Dataverse** | Business data platform with security roles, auditing, search and AI grounding | Agents and Copilot grounded on Dataverse tables |

Availability varies by product, region and licence; some features are Preview. Check the Power Platform admin centre and release plans.

### Administration

| Area | Control |
|---|---|
| **Copilot hub** (Power Platform admin centre → Copilot) | Usage and cost insights, what's new, settings by product, control who can use AI features in model-driven apps (Preview) |
| **Environment settings** | Enable/disable Copilot features and preview models per environment |
| **Data movement across geographies** | Some Copilot features require allowing data to move to regions where Azure OpenAI capacity exists; decide per policy |
| **Managed environments** | Sharing limits, usage insights, pipelines, solution checker enforcement |
| **DLP policies** | Classify connectors (business, non-business, blocked), including AI and HTTP connectors |
| **Licensing and capacity** | AI Builder credits / Copilot credits; Dynamics 365 licences include certain Copilot features |

### Where Power Platform meets Azure

| Integration | Use |
|---|---|
| **Foundry models in prompts** | Use a specific or fine-tuned model in Power Apps, Power Automate and Copilot Studio prompts |
| **Azure OpenAI / Foundry connectors and custom connectors** | Call Foundry agents or models from flows and apps |
| **Azure Functions and API Management** | Custom connectors and MCP servers for complex logic |
| **Azure AI Search** | Knowledge for Copilot Studio agents |
| **VNet support** | Azure subnet delegation so Power Platform calls private Azure resources (Key Vault, SQL, APIs via private endpoints) without public exposure; plan around 25–30 IPs per production environment |
| **Link to Microsoft Fabric** | Dataverse data available in Fabric for analytics and AI without ETL |
| **Azure Synapse / Data Lake export** | Older pattern for Dataverse export (prefer Fabric link for new work) |
| **Entra** | Identity for users, connections and agents; Conditional Access |
| **Purview and Defender** | Data governance and security across Dataverse and Azure |

## How it works on Azure

Reference pattern: invoice processing combining AI Builder, a Foundry model and Azure services.

```text
Email with PDF ─► Power Automate flow
                    ├─ AI Builder invoice processing (extract fields)
                    ├─ Prompt (Foundry model) classifies spend category and flags anomalies (JSON output)
                    ├─ Custom connector ─► APIM ─► Azure Function (validate supplier against ERP) [via VNet support]
                    ├─ Dataverse: create invoice record; approval in Teams for exceptions
                    └─ Link to Fabric: analytics on spend and exceptions
```

Prompt tool definition (conceptual):

```text
Instruction: Classify the invoice into one of: IT, Facilities, Travel, Professional Services, Other.
Flag "anomaly": true if the amount exceeds the supplier's 12-month average by more than 50%.
Inputs: {InvoiceText}, {SupplierAverage}
Output format: JSON { "category": string, "anomaly": boolean, "reason": string }
Model: <Foundry deployment connected via the model picker>
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Built-in Copilot vs custom** | Use Dynamics 365 Copilot features and agents first; build custom only for gaps. |
| **AI Builder vs Foundry** | AI Builder for document processing and simple models inside flows; Foundry for custom models, complex RAG or agents shared across channels. |
| **Data movement consent** | Some Copilot features need cross-geo processing; align with data residency policy (Topic 23 file 07). |
| **Private connectivity** | VNet support for any flow touching private Azure resources. |
| **Environment strategy** | Dev/test/prod managed environments with pipelines; Copilot features enabled per environment. |
| **Cost** | Track AI Builder/Copilot credits per environment; Azure costs for Foundry and Functions separately. |

## Hands-on

1. In a Power Apps developer environment, use Copilot to create an app from a description; review the generated tables.
2. Build a Power Automate flow with AI Builder invoice processing.
3. Create a prompt using a Foundry model deployment and call it from the flow.
4. Configure VNet support for the environment and call an Azure Function on a private endpoint.
5. In the Power Platform admin centre Copilot hub, review usage and adjust a Copilot setting for one environment.

## Security, cost and reliability notes

- **Dataverse security roles** govern what Copilot and agents can see; review them before enabling AI features.
- **Connections** used by flows and agents act with their owner's rights; use service accounts or managed identities where supported.
- **DLP** must cover AI connectors and HTTP; block consumer AI connectors where policy requires.
- **Prompt outputs** are probabilistic; validate JSON and add human approval for financial actions.
- **Capacity**: AI Builder and Copilot credits can run out; monitor and alert.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building a custom agent where Dynamics 365 already ships one | Check built-in agents and Copilot first |
| Public endpoints on Azure APIs for Power Platform | VNet support with private endpoints |
| Ignoring data-movement settings | Decide explicitly per policy |
| Flows running under a departed employee's connection | Service connections and ownership reviews |
| No DLP for AI connectors | Include them in DLP policies |

## Check yourself

1. Name four forms of AI in Power Platform and Dynamics 365.
2. What does AI Builder provide?
3. How can Foundry models be used in Power Platform?
4. What does VNet support do, and how many IPs should you plan for a production environment?
5. Which admin controls govern Copilot in Power Platform?
6. How does Dataverse data reach Fabric?

<details>
<summary>Answers</summary>

1. Copilot in maker tools, Copilot in apps for end users, AI Builder models and prompts, agents (Dynamics 365 and Copilot Studio), all grounded on Dataverse.
2. Prebuilt and custom AI models (document processing, classification, sentiment and more) used in Power Apps and Power Automate.
3. By connecting Foundry model deployments in the prompt tool's model picker (Power Apps, Power Automate, Copilot Studio), or calling Foundry via connectors.
4. Azure subnet delegation so Power Platform makes outbound calls to private resources over your network; plan about 25–30 IPs per production environment.
5. Copilot hub settings by product and environment, who can use AI in model-driven apps, data movement settings, managed environments, DLP policies, licensing/capacity.
6. Through Link to Microsoft Fabric (preferred over older Synapse/Data Lake export).

</details>

## References

- [Overview of AI Builder](https://learn.microsoft.com/ai-builder/overview)
- [Copilot hub in the Power Platform admin centre](https://learn.microsoft.com/power-platform/admin/copilot/copilot-hub)
- [Azure VNet support for Power Platform](https://learn.microsoft.com/power-platform/admin/vnet-support-overview)
- [Bring your own model for prompts](https://learn.microsoft.com/microsoft-copilot-studio/bring-your-own-model-prompts)
- [Copilot in Dynamics 365 Sales](https://learn.microsoft.com/dynamics365/sales/copilot-overview)
- [Managed environments](https://learn.microsoft.com/power-platform/admin/managed-environment-overview)
