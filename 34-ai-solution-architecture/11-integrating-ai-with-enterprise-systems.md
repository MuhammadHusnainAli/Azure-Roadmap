# Integrating AI with Enterprise Systems

## Why it matters

An AI assistant that can only chat is a novelty; one that can check an order in SAP, update a case in ServiceNow, read an opportunity in Salesforce and create a purchase requisition in Dynamics 365 changes how work gets done. Most enterprise value from agents comes from **integration with systems of record**, which is also where most risk lies: writing to ERP, exposing customer data, bypassing business rules and permissions. Architects must expose enterprise capabilities as safe, well-described tools (APIs and MCP servers), enforce identity and authorisation, use the right integration platform (API Management, Logic Apps, Power Platform connectors, Fabric), and design for reliability when back-end systems are slow or down.

## Key concepts

### Integration options

| Option | Description | Fit |
|---|---|---|
| **API Management in front of system APIs** | Expose and govern REST/OData APIs from SAP, Dynamics, custom LOB systems; convert REST APIs to **MCP servers** in APIM; catalogue in **API Center** | Primary pattern for agent tools; consistent security, throttling, logging |
| **Logic Apps (and connectors)** | 1,000+ connectors (SAP, ServiceNow, Salesforce, Dynamics 365, Oracle, SQL); workflows; can be exposed as agent tools | Integration workflows, connectors to SaaS/on-premises (via data gateway or VNet) |
| **Power Platform connectors** | Same connector ecosystem for Copilot Studio agents | Low-code agents (Topic 24) |
| **Custom MCP servers** | Domain-specific tools on Functions/Container Apps (Topic 19 file 05) | Rich, curated tool surfaces for agents |
| **Copilot connectors (Microsoft Graph)** | Bring external data into Microsoft 365 Copilot (synced or federated via MCP) | Knowledge for Copilot users |
| **Fabric mirroring/shortcuts** | Replicate operational data (SAP, Dynamics via Link to Fabric, Snowflake) for analytics and AI grounding | Read-heavy analytics and RAG over business data |
| **Event-driven integration** | Events from systems (Event Grid, Service Bus, SAP events) trigger agents | Proactive/autonomous agents |

### System-specific notes

| System | Integration notes |
|---|---|
| **SAP** (ECC, S/4HANA, BTP) | OData/BAPI/RFC via APIM, Logic Apps SAP connector (with on-premises data gateway or VNet), SAP integration platforms; SAP data in Fabric for analytics; respect SAP authorisations (principal propagation where possible) |
| **Dynamics 365** | Dataverse Web API, finance and operations data entities/OData; native connectors; Copilot features in Dynamics (Topic 24 file 11) |
| **ServiceNow** | REST APIs and connectors for incidents, requests, knowledge; common for IT and HR agents |
| **Salesforce** | REST APIs and connectors; identity federation with Entra |
| **Custom LOB/legacy** | Wrap with APIs (APIM, Functions); anti-corruption layer; consider RPA only as last resort |

### Designing tools for agents

| Principle | Detail |
|---|---|
| **Task-oriented tools** | `get_order_status(order_id)` rather than raw CRUD on every table |
| **Clear descriptions and schemas** | Precise names, parameter descriptions, enums; agents choose tools from descriptions |
| **Least privilege** | Read-only by default; separate write tools with approvals |
| **Identity** | On-behalf-of user for user-scoped data; agent identity for system tasks; never shared admin credentials |
| **Validation** | Server-side validation of every argument; business rules enforced in the API |
| **Idempotency** | Idempotency keys for writes to prevent duplicates on retries |
| **Bounded outputs** | Return minimal fields, paginated; avoid dumping large records into context |
| **Errors** | Clear, actionable error messages for the agent; no stack traces or secrets |

### Reliability with slow or fragile back ends

Timeouts, retries with backoff, circuit breakers (APIM), caching for reference data, async patterns (queue a request and notify when done) for long-running ERP transactions, and graceful degradation in the agent ("I can't reach the order system right now; I've created a ticket").

## How it works on Azure

Reference integration architecture:

```text
Agents (Foundry / Copilot Studio)
  └─ tools → API Management (Entra auth, OBO, rate limits, logging, MCP endpoints)
        ├─ SAP S/4HANA OData (via private connectivity / gateway), principal propagation
        ├─ Logic Apps workflows (ServiceNow incident creation, Salesforce updates)
        ├─ Dynamics 365 Dataverse Web API
        └─ Custom MCP server (Container Apps) wrapping legacy order system
  └─ knowledge → Foundry IQ over Fabric (mirrored SAP/Dynamics data) + document indexes
Events: Service Bus/Event Grid from ERP → autonomous agent workflows with approvals
Governance: API Center catalogue of approved tools; Agent 365/Control Plane registration
```

Expose an existing REST API as an MCP server in APIM (portal: APIs → MCP servers → expose an API as MCP server, select operations), then add it to a Foundry agent as an MCP tool with approvals for write operations.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Direct API vs workflow** | Direct API calls for simple reads/writes; Logic Apps/workflows for multi-step, connector-heavy processes. |
| **Real-time vs replicated data** | Real-time APIs for transactions and current status; replicated data in Fabric for analytics and broad RAG. |
| **MCP vs OpenAPI tools** | MCP for reusable tool servers across agents/platforms; OpenAPI tools for straightforward APIs. |
| **Write actions** | Human approval or business-rule gates; audit every write. |
| **Ownership** | System owners own APIs and tool contracts; AI teams consume via catalogue. |

## Hands-on

1. Expose a REST API through APIM as an MCP server and use it from a Foundry agent.
2. Build a Logic Apps workflow creating a ServiceNow (or test system) incident and expose it as a tool.
3. Implement on-behalf-of auth from the agent to a Dataverse or custom API.
4. Design tool contracts (names, schemas, descriptions) for an order management agent.
5. Simulate back-end outage and verify graceful degradation.

## Security, cost and reliability notes

- **Confused deputy risk**: agents must act with the user's permissions where data is user-scoped.
- **Audit**: log tool calls with identity, parameters (redacted) and results.
- **Cost**: connector/action pricing (Logic Apps), APIM tier, back-end license impacts (API call limits in SaaS systems).
- **Rate limits** of SaaS APIs (Salesforce, ServiceNow) must be respected; throttle in APIM.

## Common mistakes

| Mistake | Fix |
|---|---|
| Giving agents generic database access | Task-oriented tools |
| Shared service accounts with admin rights | OBO and least privilege |
| Unvalidated write tools | Server-side validation and approvals |
| No catalogue of tools | API Center and governance |
| Ignoring SaaS API limits | Throttling and caching |

## Check yourself

1. List the main integration options for AI with enterprise systems.
2. What principles make good agent tools?
3. How should identity flow from agents to systems of record?
4. When should you use replicated data (Fabric) instead of live APIs?
5. How does APIM help expose tools?
6. How do you handle slow or unavailable back ends?

<details>
<summary>Answers</summary>

1. APIM-fronted APIs (including MCP), Logic Apps/Power Platform connectors, custom MCP servers, Copilot connectors, Fabric mirroring/shortcuts, event-driven integration.
2. Task-oriented, clearly described, least-privilege, correct identity, server-side validation, idempotent, bounded outputs, clear errors.
3. On-behalf-of the user for user-scoped data; agent identity for system tasks; no shared admin credentials.
4. For analytics and broad grounding over business data; live APIs for transactions and current status.
5. Central auth, OBO, throttling, logging, REST-to-MCP exposure and cataloguing in API Center.
6. Timeouts, retries, circuit breakers, caching, async patterns and graceful degradation messages.

</details>

## References

- [Expose a REST API as an MCP server in API Management](https://learn.microsoft.com/azure/api-management/export-rest-mcp-server)
- [Azure API Center](https://learn.microsoft.com/azure/api-center/overview)
- [Logic Apps overview](https://learn.microsoft.com/azure/logic-apps/logic-apps-overview)
- [SAP connector for Logic Apps](https://learn.microsoft.com/azure/logic-apps/connectors/sap)
- [Dataverse Web API](https://learn.microsoft.com/power-apps/developer/data-platform/webapi/overview)
- [ServiceNow connector](https://learn.microsoft.com/connectors/service-now/)
- [Integration architecture guidance](https://learn.microsoft.com/azure/architecture/integration/integration-start-here)
