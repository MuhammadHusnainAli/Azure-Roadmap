# Work IQ, Fabric IQ and Web Grounding

## Why it matters

Not all knowledge lives in documents you index. Much of what an agent needs is **how work happens** (emails, meetings, chats, files in Microsoft 365), **the state of the business** (sales, inventory, KPIs in Fabric) and **what is happening in the world right now** (the public web). Copying all of that into a search index is slow, expensive and risky for permissions. Microsoft's answer is a family of grounding layers: **Work IQ** for Microsoft 365, **Fabric IQ** for business data, and **web grounding** tools for live public content, alongside **Foundry IQ** (file 08) for enterprise documents. Architects must know which layer answers which question, what each costs and licenses, and where data leaves your compliance boundary.

## Key concepts

### Microsoft IQ at a glance

| Layer | Grounds agents in | Built on | Typical question |
|---|---|---|---|
| **Foundry IQ** | Enterprise content (Blob, SharePoint, OneLake, SQL, indexes) | Azure AI Search agentic retrieval | "What does our travel policy say about rail upgrades?" |
| **Work IQ** | Microsoft 365 work context | Microsoft 365 Copilot and Graph | "What did we agree with Contoso in last week's meetings?" |
| **Fabric IQ** | Business data and meaning | OneLake, Power BI semantic models, ontologies, graph, data agents | "Which shipments breached cold-chain limits this month and why?" |
| **Web grounding** | Live public web | Bing (Web Search, Grounding with Bing tools) | "What changed in the EU AI Act guidance this week?" |

Microsoft's Fabric documentation also mentions **Web IQ** as part of the Microsoft IQ family; in Foundry Agent Service, web grounding is delivered through the web tools below.

### Work IQ

Work IQ is a **workplace intelligence layer** that combines Microsoft 365 data with contextual understanding. It can reason over email, meetings and calendars, OneDrive and SharePoint documents, Teams messages, people and organisational context, Planner, and enterprise search. Access is **permission-trimmed and policy-enforced** automatically: no vector stores or sync jobs to run.

| Protocol | Use when | Caller |
|---|---|---|
| **A2A** (v1.0 and v0.3) | Another agent delegates a task to Work IQ | Agent |
| **REST** | Your app or backend calls Work IQ conversationally | Service |
| **MCP** (local and remote) | An LLM client uses Work IQ as tools | Assistant, IDE, agent |

The **Work IQ MCP server** exposes a small fixed set of generic tools that act on Microsoft 365 resource paths ("fewer tools, more paths"):

| Category | Tools |
|---|---|
| Entity | `fetch`, `fetch_blob`, `create_entity`, `update_entity`, `delete_entity`, `do_action`, `call_function` |
| Copilot | `ask`, `retrieve` (Preview), `list_agents` |
| Schema | `get_schema`, `search_paths` |

Authentication is Microsoft Entra ID (discovered via `/.well-known/oauth-protected-resource`). Licensing: the Work IQ API is billed on a usage basis with **Copilot Credits**; admins can allow or block Work IQ MCP servers in the Microsoft 365 admin centre. Because Work IQ can also **write** (send mail, create events), treat it as an action surface, not just a knowledge source.

### Fabric IQ

Fabric IQ provides context on **the state of the business**, in business language rather than table names. Three layers:

| Layer | Item | Role |
|---|---|---|
| Unified data | **OneLake** (shortcuts, mirroring, catalogue) | One governed data estate |
| Business intelligence | **Power BI semantic models** | Trusted measures, hierarchies, KPIs |
| Operational intelligence | **Ontology** (Preview), **Graph** | Entity types, relationships, rules and metrics bound to live data |

Agent-facing items include **Fabric data agents** (natural-language analysts over semantic models, lakehouses, warehouses and ontologies; publishable to Microsoft 365, Foundry, Copilot Studio and custom apps) and **operations agents** (monitor real-time data and recommend actions). See [Topic 12 file 07](../12-data-and-analytics/07-fabric-iq-and-data-agents.md) for the Fabric side.

### Web grounding in Foundry Agent Service

| Tool | Status | Bing resource | Notes |
|---|---|---|---|
| **Web Search** (recommended) | GA | Managed by Microsoft | Simplest; `user_location`, `search_context_size`; domain allow/block lists via a Bing Custom Search instance; Azure OpenAI models |
| **Grounding with Bing Search** | GA | You create it | `count`, `freshness`, `market`, `set_lang`; also supports non-OpenAI models sold by Azure |
| **Grounding with Bing Custom Search** | Preview | You create it | Restrict to specific domains |

Critical compliance facts from Microsoft Learn:

- These are **First Party Consumption Services** governed by the Grounding with Bing terms of use.
- The **Microsoft Data Protection Addendum doesn't apply**, and data **flows outside the Azure compliance and geo boundary**; elevated Government Community Cloud commitments are waived.
- Web grounding tools **don't respect VPNs or private endpoints**; they act as public endpoints even in network-secured Foundry projects.
- Usage incurs extra cost, and admins can restrict access.

In agentic retrieval, the **Web** knowledge source uses Bing and requires answer synthesis (file 07).

## How it works on Azure

| Integration | How |
|---|---|
| Work IQ in a Foundry agent | Work IQ tool in the Foundry tool catalogue (also usable via toolboxes), or the Work IQ MCP server; Work IQ knowledge source in a Foundry IQ knowledge base (Preview) |
| Fabric IQ in a Foundry agent | Fabric IQ / Fabric data agent tools in the catalogue; Fabric Data Agent and Fabric Ontology knowledge sources (Preview) |
| Web in a Foundry agent | `WebSearchTool` on a prompt agent; `WebSearchToolboxTool` in a toolbox for hosted agents |
| Copilot Studio | Work IQ MCP servers and Foundry IQ connections ([Topic 24](../Roadmap.md)) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Index Microsoft 365 content or use Work IQ?** | Use Work IQ (or remote SharePoint) when users have the right licences and you want live permissions without copies. Index SharePoint into Azure AI Search when you need custom ranking, non-M365 users, or GA-only components. |
| **Fabric IQ or text-to-SQL?** | Prefer Fabric data agents and semantic models for governed metrics; ad hoc text-to-SQL risks inconsistent definitions (file 11). |
| **Web grounding or curated content?** | Web for current events and public facts; curated indexes for authoritative internal answers. Restrict domains where possible. |
| **Regulated workloads** | Web grounding sends query data outside the Azure compliance boundary; many regulated or government workloads must disable it. |
| **Write-capable tools** | Work IQ entity tools can send mail or change data; require approvals and least-privilege scopes. |
| **Cost model** | Foundry IQ: search tokens and tier; Work IQ: Copilot Credits; web: per-call Bing pricing; Fabric: capacity units. Budget each separately. |

## Hands-on

Add web grounding to a Foundry prompt agent and observe citations.

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, WebSearchTool, WebSearchApproximateLocation

with AIProjectClient(endpoint="https://<acct>.services.ai.azure.com/api/projects/<proj>",
                     credential=DefaultAzureCredential()) as project:
    agent = project.agents.create_version(
        agent_name="news-analyst",
        definition=PromptAgentDefinition(
            model="gpt-5-mini",
            instructions="Answer using web results only. Always cite URLs. Say when sources disagree.",
            tools=[WebSearchTool(user_location=WebSearchApproximateLocation(country="GB", city="London"))]))

    with project.get_openai_client(agent_name=agent.name) as oai:
        r = oai.responses.create(tool_choice="required",
                                 input="What did Microsoft announce for Foundry at Ignite this year?")
        print(r.output_text)
        for item in r.output:
            for c in getattr(item, "content", []) or []:
                for a in getattr(c, "annotations", []) or []:
                    print(a.type, getattr(a, "url", ""))
```

Then, as a design exercise, write a one-page grounding map for a fictional company: list ten typical employee questions, and for each name the layer (Foundry IQ, Work IQ, Fabric IQ or web), the licence or cost implication, and whether data leaves the compliance boundary.

## Security, cost and reliability notes

- **Data boundary.** Web grounding: outside the Azure compliance boundary, no DPA coverage, ignores private endpoints. Document this in your data protection impact assessment.
- **Permissions.** Work IQ and remote SharePoint enforce Microsoft 365 permissions per user; that only works if the agent passes the **user's** identity (on-behalf-of), not an app identity ([Topic 18 file 13](../18-agents-with-foundry-agent-service/13-agent-identity-and-authorisation.md)).
- **Admin controls.** Azure admins can block web grounding tools; Microsoft 365 admins can block Work IQ MCP servers.
- **Write actions.** Gate Work IQ write tools behind human approval.
- **Preview status.** Several connectors (Work IQ `retrieve`, Fabric ontology, Fabric and Work IQ knowledge sources) are in preview; check status before production commitments.

## Common mistakes

| Mistake | Fix |
|---|---|
| Copying all of Microsoft 365 into a vector index | Use Work IQ or remote SharePoint where licensing allows |
| Enabling web grounding in a regulated tenant without review | Check the data boundary and DPA implications first |
| Assuming private endpoints cover web grounding | They don't; web tools are public endpoints |
| Text-to-SQL over raw tables for KPIs | Ground on semantic models, ontologies and data agents |
| Using an app identity for Work IQ | Use the signed-in user's delegated identity |
| No domain restrictions on web search for a support bot | Use allow-lists |

## Check yourself

1. Which grounding layer answers questions about meetings and emails, and which about KPIs?
2. Which protocols does Work IQ support, and when would you use each?
3. Describe the Work IQ MCP design principle "fewer tools, more paths".
4. How is the Work IQ API billed?
5. What are the three layers of Fabric IQ?
6. Compare Web Search and Grounding with Bing Search.
7. What three compliance facts apply to web grounding tools?
8. Why must Work IQ calls carry the user's identity?

<details>
<summary>Answers</summary>

1. Work IQ for meetings and email; Fabric IQ for KPIs and business data.
2. A2A for agent delegation, REST for apps and services calling programmatically, MCP for LLM clients using Work IQ as tools.
3. A fixed set of generic tools operates on resource paths; new workloads add paths rather than new tools, and schemas are discovered at runtime.
4. Usage-based billing with Copilot Credits.
5. Unified data (OneLake), business intelligence (Power BI semantic models), operational intelligence (ontologies and graph).
6. Both GA; Web Search needs no Bing resource and suits Azure OpenAI models; Grounding with Bing Search needs your own Bing resource, offers count/freshness/market/language parameters, and supports non-OpenAI models sold by Azure.
7. Data flows outside the Azure compliance and geo boundary; the Data Protection Addendum doesn't apply; the tools don't respect VPNs or private endpoints.
8. So Microsoft 365 permissions are enforced for that user; an app identity could expose content the user can't see.

</details>

## References

- [Work IQ API overview](https://learn.microsoft.com/microsoft-365/copilot/extensibility/work-iq/api-overview)
- [Work IQ MCP overview](https://learn.microsoft.com/microsoft-365/copilot/extensibility/work-iq/mcp/overview)
- [What is Fabric IQ?](https://learn.microsoft.com/fabric/iq/overview)
- [Web grounding tools overview (Foundry Agent Service)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/web-overview)
- [Web Search tool](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/web-search)
- [What is Foundry IQ? (relationship to Work IQ and Fabric IQ)](https://learn.microsoft.com/azure/foundry/agents/concepts/what-is-foundry-iq)
