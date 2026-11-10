# 11 · Integration, Messaging and APIs

> Tracks: E ● A ● C ○ · Part B · Core Infrastructure and Application Platform

Applications and AI systems rarely live alone. They exchange messages, react to events, stream telemetry, call each other's APIs and move data in bulk. This topic covers Azure's integration services: the four messaging services (Service Bus, Event Grid, Event Hubs, Storage Queues), Logic Apps for workflow and B2B integration (now with agent loops), API Management as the API and **AI gateway**, API Center as the inventory for APIs and **MCP servers**, and Data Factory for bulk data integration. It finishes with the architecture patterns that make distributed systems reliable.

## What you will be able to do

- Choose between Service Bus, Event Grid, Event Hubs and Storage Queues from message semantics, not familiarity.
- Build reliable messaging with Service Bus sessions, dead-lettering, duplicate detection and geo-replication.
- Route Azure and application events with Event Grid push and pull delivery, and connect MQTT clients.
- Stream high-volume events with Event Hubs partitions, consumer groups, the Kafka endpoint and Capture.
- Pick Logic Apps Consumption or Standard, and build an agentic workflow with an agent loop and tools.
- Choose an API Management tier, write policies, and federate API ownership with workspaces.
- Run APIM as an AI gateway: token limits, priority load balancing with circuit breakers, semantic caching, token metrics and MCP server governance.
- Build an API and MCP server inventory with API Center.
- Move data in bulk with Data Factory pipelines and integration runtimes.
- Apply the outbox, saga, claim check, competing consumers and load-levelling patterns.

## Prerequisites

[Topic 02](../02-identity-and-access/README.md) (Entra ID, managed identities, RBAC), [Topic 05](../05-networking/README.md) (private endpoints, VNets), [Topic 09](../09-databases/README.md) (Cosmos DB, SQL) and [Topic 10](../10-application-platform/README.md) (Functions, Durable Functions, App Service).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Messaging Decision Guide](01-messaging-decision-guide.md) | 35 min | CLI |
| 02 | [Azure Service Bus](02-azure-service-bus.md) | 60 min | CLI, Bicep, Python |
| 03 | [Azure Event Grid](03-azure-event-grid.md) | 50 min | CLI, Python |
| 04 | [Azure Event Hubs](04-azure-event-hubs.md) | 55 min | Bicep, Python, Kafka |
| 05 | [Azure Logic Apps](05-azure-logic-apps.md) | 55 min | Portal, VS Code |
| 06 | [Azure API Management](06-azure-api-management.md) | 60 min | CLI, Bicep, policies |
| 07 | [API Management as AI Gateway](07-api-management-as-ai-gateway.md) | 60 min | Bicep, policies |
| 08 | [Azure API Center](08-azure-api-center.md) | 35 min | CLI |
| 09 | [Azure Data Factory for Integration](09-azure-data-factory-for-integration.md) | 40 min | Portal, CLI |
| 10 | [Integration Architecture Patterns](10-integration-architecture-patterns.md) | 50 min | Python, exercise |

**AI engineers:** 01, 02, 03, 07 and 10 in depth; 04 and 06 for working knowledge. **Architects:** all of it, with 01, 06, 07 and 10 in depth. **Consultants:** 01, 05, 06, 07 and 08.

## Completion checklist

- [ ] I can explain the difference between events and messages and pick the right messaging service.
- [ ] I have sent and received Service Bus messages with sessions and inspected a dead-letter queue.
- [ ] I can explain Service Bus geo-replication vs geo-disaster recovery.
- [ ] I have subscribed to `BlobCreated` events with CloudEvents schema and dead-lettering.
- [ ] I can explain when to use Event Grid pull delivery and the MQTT broker.
- [ ] I can size Event Hubs partitions and consumer groups, and I have checkpointed a consumer.
- [ ] I can compare Logic Apps Consumption and Standard, and I have built an agent loop with tools.
- [ ] I can choose an APIM tier for a given networking requirement.
- [ ] I have applied `llm-token-limit`, a priority backend pool with a circuit breaker, and `llm-emit-token-metric`.
- [ ] I have exposed an API as an MCP server through APIM and registered it in API Center.
- [ ] I have built a parameterised Data Factory copy pipeline with managed identity.
- [ ] I can draw the transactional outbox and an orchestrated saga with compensation.

## Maps to

- **AI-200**: develop message-based and event-based solutions (Service Bus, Event Grid, Event Hubs); implement API Management.
- **AZ-305**: design integration and messaging solutions, API integration, data integration.
- **AI-103**: secure and govern model access through an AI gateway; host and govern MCP tools.
- **AZ-104**: working knowledge only.

## Next

[12 · Data and Analytics](../12-data-and-analytics/README.md)
