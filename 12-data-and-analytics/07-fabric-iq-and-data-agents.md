# Fabric IQ and Data Agents

> Topic: 12 · Data and Analytics · Tracks: E ● A ● C ● · Status checked: 2026-11

## Why it matters

Agents are only as good as the context they reason over. Documents give them policies and procedures; structured business data tells them what is actually happening (orders, shipments, customers, assets). **Fabric IQ** is Microsoft's layer for giving agents and people that business context, grounded in OneLake. It combines **semantic models** (trusted KPIs), **ontologies** (a shared, machine-understandable model of the business) and **graph**. **Fabric data agents** turn that context into natural-language Q&A that other agents (in Foundry, Copilot Studio and Microsoft 365 Copilot) can call. For AI architects this is how you build an agent that answers "why are cold-chain breaches up in the north region?" correctly and with permissions enforced.

## Key concepts

### Microsoft IQ: where Fabric IQ fits

Microsoft groups its "intelligence layers" under **Microsoft IQ**:

| Layer | Provides context about | Built on |
|---|---|---|
| **Work IQ** | How people work: email, meetings, files, chats | Microsoft 365 / Microsoft Graph |
| **Foundry IQ** | Policies, procedures, authoritative documents (knowledge bases for retrieval) | Azure AI Search (topic 17) |
| **Fabric IQ** | Business entities and data: state of the business | OneLake, semantic models, ontology |
| **Web IQ** | The public web | Web grounding |

### The three layers of Fabric IQ

| Layer | Delivered by | Role |
|---|---|---|
| **Unified data** | OneLake (shortcuts, mirroring, catalogue) | One governed copy of analytical, real-time and operational data |
| **Business intelligence** | **Power BI semantic models** | Curated measures, hierarchies and relationships: trusted KPIs |
| **Operational intelligence** | **Ontology** (Preview) | Entity types, properties, relationships, rules and metrics bound to live data, so agents can reason and act |

### Ontology (Preview, checked 2026-11)

An **ontology** is a shared, machine-understandable vocabulary of the business:

| Element | Example |
|---|---|
| **Entity types** | Customer, Order, Shipment, Truck, Sensor |
| **Properties** | Shipment.status, Sensor.temperature |
| **Relationships** | Order *has* Shipment; Shipment *carried by* Truck; Truck *has* Sensor |
| **Data bindings** | Each entity type and property mapped to lakehouse, eventhouse or semantic model data |
| **Rules** | Business logic in natural language, with condition → action through **Activator** ("if a cold-chain shipment exceeds 8°C for 10 minutes, alert the dispatcher") |
| **Metrics** | Source-owned DAX measures carried over from semantic models |

You can create an ontology from scratch, **generate it from an existing semantic model** (bootstrapping from trusted logic), generate details with the built-in **ontology agent**, or import from RDF/OWL. A built-in **graph** (powered by Graph in Fabric, which must be enabled in tenant settings) lets agents traverse relationships (Order → Shipment → Sensor → Breach). Ontology data bindings may need refreshing to pick up upstream changes. Check current behaviour.

### Fabric IQ workload items

| Item | Status (2026-11) | Use |
|---|---|---|
| Ontology | Preview | Shared business vocabulary and rules |
| Power BI semantic model | GA | Trusted KPIs |
| Graph | Preview | Relationship-heavy questions (paths, impact chains) |
| Planning | Check current status | Planning and forecasting on semantic models |
| **Data agent** | **GA** | Natural-language Q&A over Fabric data |
| Operations agent | Preview | Monitors real-time data and recommends or triggers actions |

### Fabric data agents

A **data agent** is a configurable conversational analyst over Fabric data:

| Aspect | Detail (checked 2026-11) |
|---|---|
| Data sources | Up to **five**, in any mix of lakehouse, warehouse, Power BI semantic model, KQL database (Eventhouse), mirrored database, ontology, and Microsoft Graph |
| How it answers | Chooses a source, then generates **read-only** queries: NL2SQL (lakehouse and warehouse), NL2DAX (semantic models), NL2KQL (KQL databases), Graph queries. Executes them and summarises results |
| Configuration | Select tables; **agent instructions** (routing rules, terminology); **example queries** (up to 100 per source; not supported for semantic models) |
| Identity | Runs with the **requesting user's identity and permissions**. RLS and CLS apply, Purview DLP and access policies are respected, and sharing an agent doesn't grant data access |
| Requirements | Paid **F2 or higher** capacity (or P1+ with Fabric enabled); data agent tenant settings; cross-geo processing and storage for AI where required |
| Limits | Read-only; no unstructured files (PDF, DOCX); English only; LLM not selectable; responses capped at 25 rows × 25 columns; data source and agent must be in the same region |
| ALM | Git integration, deployment pipelines, diagnostics |

### Consuming data agents

| Consumer | How |
|---|---|
| Fabric users | Chat with the published agent in Fabric |
| **Microsoft Foundry Agent Service** | Add the **Fabric data agent tool** to a Foundry agent through a project connection (workspace ID and artifact ID of the published data agent). The call runs with the **end user's identity** (on-behalf-of); service principal authentication isn't supported. A **Fabric IQ** tool in Foundry can also connect agents to ontology context (check current status) |
| **Copilot Studio** | Add the Fabric data agent as a connected agent or knowledge source |
| **Microsoft 365 Copilot** | Publish to Microsoft 365 Copilot so users can ask in Teams and Copilot Chat |
| Custom apps | Through the published endpoint / SDK (check current API options) |

## How it works on Azure

### Architecture: multi-agent with structured and unstructured knowledge

```text
User (Teams / custom app, Entra sign-in)
   │
Foundry agent "Operations assistant"
   ├─ Foundry IQ knowledge base (SOPs, contracts, policies in AI Search)   ← "what should we do?"
   ├─ Fabric data agent tool (OBO user identity)                          ← "what is happening?"
   │     ├─ semantic model "Logistics KPIs" (NL2DAX)
   │     ├─ eventhouse "Telemetry" (NL2KQL)
   │     └─ ontology "Supply chain" (entities, relationships, rules)
   └─ Action tools (Logic Apps / Functions via MCP) with approval          ← "do it"
```

### Build a data agent

1. Prepare data: gold tables with clear names, descriptions on semantic model tables and measures, KQL functions for common questions.
2. Create **Data agent** in a workspace on an F2+ capacity and add up to five sources; select only the relevant tables.
3. Write **agent instructions**, for example:

   ```text
   - Questions about revenue, margin or targets → use the "Sales KPIs" semantic model.
   - Questions about live truck telemetry or the last 24 hours → use the Telemetry KQL database; always filter by time.
   - "Region" means the SalesRegion column, not the shipping region.
   - Fiscal year starts 1 July.
   ```
4. Add **example queries** (question → SQL or KQL) for tricky questions.
5. Test, check **diagnostics** to see the generated queries, then **publish**.
6. Connect from Foundry: create a project connection to the published agent and add the Fabric tool to your agent.

### Calling from a Foundry agent (sketch)

```python
# Pseudocode: tool and class names follow the current azure-ai-projects / agents SDK; check the Fabric tool docs.
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

project = AIProjectClient(endpoint="https://<resource>.services.ai.azure.com/api/projects/<project>",
                          credential=DefaultAzureCredential())
# 1) Create a connection in the project to the published Fabric data agent (workspace ID + artifact ID)
# 2) Create the agent with the Fabric data agent tool referencing that connection ID
# 3) Calls run on behalf of the signed-in end user, who needs access to the data agent and its sources
```

The exact SDK surface for the Fabric tool changes between agent API versions (topic 18 covers current code). The design point is that the user's identity flows through to Fabric.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Data agent vs RAG over exported data | Use data agents (live, governed queries) for structured facts and KPIs. Don't export tables into a vector index; numbers retrieved by similarity are unreliable |
| Semantic model vs ontology as grounding | Semantic models for KPI-centric questions (stable, GA). Add an ontology when agents must reason across domains, traverse relationships or act on rules, and accept preview status |
| One agent vs several | One data agent per domain with a focused table set is more accurate than one agent over everything. Orchestrate domain agents from Foundry |
| Identity | Data agents need user identity; design for on-behalf-of and ensure users actually have data access. They aren't suited to anonymous or service-only scenarios |
| Region | Keep the data agent, its sources and Foundry resources in aligned regions (same-region requirement for agent and sources) |
| Answer limits | 25 × 25 row and column caps make data agents a poor fit for returning datasets. Use them for insights, and use reports or exports for bulk data |

## Hands-on

1. Take the gold tables and semantic model from files 03 and 06. Add descriptions to every table, column and measure.
2. Create a data agent with the semantic model and one lakehouse; write instructions and three example SQL queries.
3. Ask ten realistic questions; use diagnostics to inspect generated DAX and SQL; refine instructions until at least eight are correct.
4. Test as a user restricted by RLS and confirm the answers are trimmed.
5. Connect the published agent to a Foundry agent as a tool and ask the same questions from the Foundry playground.
6. Optional (preview): generate an ontology from the semantic model and inspect entity types, relationships and the graph.

## Security, cost and reliability notes

- **Security:** data agents enforce the caller's permissions, so fix the data permissions first. Sharing an agent with someone who can't read the data gives them nothing, but giving broad data access to make an agent "work" is a real risk.
- **Security:** agent interactions can be captured by Purview audit and eDiscovery. Consider this for sensitive domains.
- **Security:** set tenant settings for cross-geo processing deliberately; prompts may be processed outside the capacity's geography if allowed.
- **Cost:** data agents and Copilot consume capacity CUs, and the queries they generate also consume CUs on the queried items.
- **Reliability:** answers are generated, so evaluate them. Keep a test set of questions with expected answers and re-run it when models, instructions or schemas change (topic 22).
- **Reliability:** preview items (ontology, graph, operations agent) can change. Isolate them from production-critical paths.

## Common mistakes

- Pointing a data agent at hundreds of raw tables with cryptic names.
- Expecting a data agent to read PDFs or other unstructured files.
- Using a service principal for the Foundry connection (not supported) and wondering why calls fail.
- Building an ontology before the semantic model and gold data are trustworthy.
- Skipping evaluation because "it uses our real data".

## Check yourself

1. What does Fabric IQ add on top of OneLake?
2. What is an ontology made of?
3. Which query languages can a data agent generate, and for which sources?
4. Whose identity does a data agent use when called from Foundry?
5. Name three limitations of data agents.
6. When would you add an ontology rather than rely only on a semantic model?

<details>
<summary>Answers</summary>

1. Business context: semantic models with trusted KPIs, and ontologies that define entities, relationships, rules and metrics bound to data, so people and agents interpret data in business terms.
2. Entity types, properties, relationships, data bindings, rules (with Activator actions) and metrics.
3. SQL for lakehouses and warehouses, DAX for semantic models, KQL for KQL databases (plus Microsoft Graph queries).
4. The end user's identity (on-behalf-of); service principal authentication isn't supported.
5. Read-only; no unstructured files; English only; can't change the LLM; 25-row and 25-column cap; data sources must be in the same region; up to five sources.
6. When agents must reason across domains, traverse relationships (impact chains) or apply business rules and actions, beyond KPI questions.

</details>

## References

- [What is Fabric IQ?](https://learn.microsoft.com/fabric/iq/overview)
- [What is ontology (preview)?](https://learn.microsoft.com/fabric/iq/ontology/overview)
- [Fabric data agent concepts](https://learn.microsoft.com/fabric/data-science/concept-data-agent)
- [Data agent tenant settings](https://learn.microsoft.com/fabric/data-science/data-agent-tenant-settings)
- [Consume a data agent from Microsoft Foundry](https://learn.microsoft.com/fabric/data-science/data-agent-foundry)
- [Fabric data agent tool in Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/fabric)
- [Fabric IQ tool in Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/fabric-iq)
- [Microsoft IQ](https://learn.microsoft.com/microsoft-iq)
