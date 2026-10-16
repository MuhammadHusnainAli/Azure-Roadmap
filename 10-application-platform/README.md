# 10 · Application Platform

> Tracks: E ● A ● C ○ · Part B · Core Infrastructure and Application Platform

The application platform is where most business applications and AI front ends actually run: web apps and APIs on App Service, event-driven code on Azure Functions, long-running workflows on Durable Functions, front ends on Static Web Apps, and the supporting services that make them configurable, real-time and communicative. Functions and App Service now also host MCP servers and agent tools. This topic covers these managed hosts, the services around them, and the patterns for modernising existing applications onto them.

## What you will be able to do

- Deploy and scale web apps on App Service with deployment slots, custom domains and managed certificates.
- Secure App Service with VNet integration, private endpoints and built-in authentication.
- Choose a Functions hosting plan (Flex Consumption, Premium, Dedicated, Container Apps) and control cold start.
- Build long-running, human-in-the-loop workflows with Durable Functions and the Durable Task Scheduler.
- Host remote MCP servers and agent tool back ends on Functions.
- Host front ends on Static Web Apps with linked APIs and preview environments.
- Manage configuration and feature flags centrally with App Configuration, including AI configuration.
- Add chat, voice, SMS, email and voice agents with Azure Communication Services.
- Stream model output to clients in real time with SignalR or Web PubSub.
- Choose a modernisation approach for an existing application and plan it with the strangler fig pattern.

## Prerequisites

[Topic 02](../02-identity-and-access/README.md) (app registrations, managed identities), [Topic 04](../04-tooling-and-infrastructure-as-code/README.md) (CLI, Bicep, azd), [Topic 05](../05-networking/README.md) (VNets, private endpoints, Front Door) and [Topic 07](../07-containers-and-kubernetes/README.md) (containers, Container Apps).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Azure App Service](01-azure-app-service.md) | 60 min | CLI |
| 02 | [App Service Networking and Security](02-app-service-networking-and-security.md) | 50 min | CLI |
| 03 | [Azure Functions](03-azure-functions.md) | 60 min | CLI, Python |
| 04 | [Durable Functions and Durable Task Scheduler](04-durable-functions-and-durable-task-scheduler.md) | 50 min | Python |
| 05 | [Functions for AI](05-functions-for-ai.md) | 50 min | Python |
| 06 | [Static Web Apps](06-static-web-apps.md) | 30 min | CLI |
| 07 | [Azure App Configuration](07-azure-app-configuration.md) | 40 min | CLI, Python |
| 08 | [Azure Communication Services](08-azure-communication-services.md) | 40 min | Python |
| 09 | [Azure SignalR, Web PubSub and Fluid Relay](09-azure-signalr-web-pubsub-and-fluid-relay.md) | 35 min | Python |
| 10 | [Azure Maps and Notification Hubs](10-azure-maps-and-notification-hubs.md) | 25 min | REST |
| 11 | [Application Modernisation Patterns](11-application-modernisation-patterns.md) | 45 min | Exercise |

**AI engineers:** 01, 03, 04, 05, 07 and 09 in depth. **Architects:** all of it, with 01, 02, 03 and 11 in depth. **Consultants:** 01, 03, 08 and 11.

## Completion checklist

- [ ] I can choose an App Service plan tier and explain what each adds.
- [ ] I have deployed to a staging slot and swapped it into production.
- [ ] I can explain the difference between VNet integration (outbound) and private endpoints (inbound).
- [ ] I have enabled Easy Auth with Microsoft Entra ID on an app.
- [ ] I can compare Flex Consumption, Premium and Dedicated plans for Functions.
- [ ] I have written a Durable Functions orchestration with fan-out/fan-in and an external event.
- [ ] I have built and called a remote MCP server on Azure Functions.
- [ ] I can manage a feature flag in App Configuration and read it in code.
- [ ] I can explain how to stream LLM tokens to many clients with Web PubSub.
- [ ] I can apply the strangler fig pattern to a monolith migration plan.

## Maps to

- **AZ-104**: configure App Service.
- **AI-200**: implement Azure App Service web apps and Azure Functions; Durable Functions; App Configuration.
- **AZ-305**: design application architectures and compute solutions.
- **AI-103**: host AI apps, tools and MCP servers.

## Next

11 · Integration, Messaging and APIs (see [Roadmap.md](../Roadmap.md))
