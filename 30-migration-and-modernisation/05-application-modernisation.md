# Application Modernisation

## Why it matters

Rehosting moves applications; modernising changes how they're built and run so they're cheaper to operate, easier to change and ready for AI. The barrier has always been effort: upgrading old .NET Framework or Java versions, replacing on-premises dependencies (file shares, Windows authentication, message queues, local secrets), containerising, and writing IaC and pipelines. Two developments change the economics: **Azure Migrate's agentless web app migration** to App Service at scale, and **GitHub Copilot modernization**, an agentic tool that assesses, plans, upgrades and migrates Java, .NET and C++ applications with humans reviewing every change. This file covers target platforms, modernisation patterns and how to use these tools in a programme.

## Key concepts

### Target platforms

| Target | Fit |
|---|---|
| **App Service** | Web apps and APIs (.NET, Java, Node.js, Python, PHP) with minimal change; slots, autoscale, managed TLS |
| **Container Apps** | Containerised microservices and APIs, event-driven scale (KEDA), scale to zero, Dapr; less Kubernetes overhead |
| **AKS** | Complex container platforms, custom networking/operators, large estates, portability |
| **Azure Functions** | Event-driven and background processing |
| **App Service Managed Instance** / Windows containers | Legacy Windows apps with OS dependencies (check current support) |

### Modernisation patterns

| On-premises dependency | Azure replacement |
|---|---|
| Config files with secrets | App Configuration + Key Vault references, managed identity |
| Windows authentication | Entra ID (OIDC), Entra Domain Services only if unavoidable |
| Local/SMB file storage | Azure Files, Blob Storage |
| MSMQ / on-premises queues | Service Bus |
| Scheduled tasks/Windows services | Functions, Container Apps jobs, WebJobs |
| Session state in memory | Azure Managed Redis |
| SMTP relay | Azure Communication Services |
| Logging to files | OpenTelemetry/Application Insights |

Apply the **strangler fig** pattern to modernise incrementally: route functionality piece by piece to new services behind a gateway while the legacy system shrinks.

### GitHub Copilot modernization

| Capability | Detail |
|---|---|
| **Assessment and planning** | Analyses code, configuration and dependencies; generates a plan; identifies outdated libraries and migration issues |
| **Code transformation** | Upgrades runtimes, frameworks and toolsets (.NET Framework → modern .NET, Java 8/11 → 17/21, Spring Boot upgrades, C++ toolsets) using tools such as OpenRewrite; predefined tasks for Azure migration (secrets → Key Vault, messaging → Service Bus, identity → managed identity) |
| **Build and test** | Resolves build problems, migrates and runs unit tests |
| **Security** | CVE scanning after upgrades with automated fixes in agent mode |
| **Deploy** | Generates containerisation assets, IaC and CI/CD |
| **Modernization agent (CLI)** | Portfolio-level assessment and planning across many applications |

IDE upgrade/migration experiences and the modernization agent CLI are GA. Humans stay in the loop: every change is reviewable ([Topic 24 file 07](../24-copilot-and-low-code-agents/07-github-copilot-for-azure-workloads.md)).

### Azure Migrate web app migration

Discovers ASP.NET (and Java) web apps on servers, assesses readiness for App Service/AKS, and performs **agentless ASP.NET web app migration at scale to App Service** from VMware environments. The **App Service migration assistant** handles individual apps.

## How it works on Azure

Modernisation workflow for an ASP.NET Framework application:

```text
1. Azure Migrate: discover app, assess App Service readiness, dependency analysis.
2. Decide: replatform as-is to App Service (Windows) now, or upgrade to modern .NET and containerise.
3. Copilot modernization (VS Code / Visual Studio): assessment report → upgrade plan → .NET upgrade
   → predefined tasks: connection strings → managed identity + Key Vault references; file share → Blob; MSMQ → Service Bus
   → build/test fixes → CVE scan → Dockerfile + Bicep + GitHub Actions workflow generated.
4. Review PRs; run CI with tests and security scans (Topic 27).
5. Deploy to Container Apps or App Service with slots; load test; cut over traffic via Front Door.
6. Decommission the VM; add observability and SLOs.
```

For a portfolio, the **modernization agent** (delivered through the Modernize CLI) follows an **Assess → Plan → Execute** model across many repositories, with reusable custom skills for organisation-specific patterns; follow its quickstart for the current commands.

Strangler fig with API Management:

```text
Clients ─► API Management ─► /orders/* → new Container Apps service
                          └► everything else → legacy app on VM (until migrated)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Replatform vs refactor** | Replatform to App Service quickly for low-change apps; refactor apps with active development or strategic value. |
| **App Service vs Container Apps vs AKS** | Default to App Service or Container Apps; AKS only where platform requirements justify it. |
| **Tool-assisted upgrades** | Use Copilot modernization to cut effort, but budget for review, testing and edge cases. |
| **Incremental vs big bang** | Strangler fig for large monoliths; avoid long rewrites without delivered value. |
| **Identity and secrets** | Managed identity everywhere as part of modernisation. |
| **AI readiness** | Expose capabilities as APIs and events so AI agents can use them later (MCP servers over modernised APIs). |

## Hands-on

1. Assess a sample ASP.NET app with Azure Migrate and migrate it to App Service.
2. Run GitHub Copilot modernization on a sample Java or .NET app: generate the plan, upgrade, apply the Key Vault task, review changes.
3. Containerise the upgraded app and deploy to Container Apps with the generated IaC and workflow.
4. Implement a strangler fig route in API Management for one endpoint.
5. Compare operating cost and deployment frequency before and after.

## Security, cost and reliability notes

- **Review agent changes**: generated code and IaC must pass review, tests and security scans.
- **Secrets removal** is often the biggest security win of modernisation.
- **Cost**: PaaS reduces operations cost but may change compute cost; compare TCO, not only infrastructure.
- **Reliability**: add health probes, autoscale, zones and SLOs as part of the move.

## Common mistakes

| Mistake | Fix |
|---|---|
| Rewriting everything | Incremental strangler fig |
| Containerising without fixing configuration and secrets | Externalise config, managed identity |
| Choosing AKS by default | App Service/Container Apps first |
| Accepting AI-generated changes without tests | Review, test, scan |
| No observability after modernisation | OpenTelemetry and SLOs |

## Check yourself

1. Compare App Service, Container Apps and AKS as modernisation targets.
2. Map five on-premises dependencies to Azure replacements.
3. What can GitHub Copilot modernization do?
4. What does Azure Migrate offer for web apps?
5. What is the strangler fig pattern?
6. Why must humans review modernisation changes?

<details>
<summary>Answers</summary>

1. App Service: simplest for web apps/APIs; Container Apps: containerised microservices with event scaling and less overhead; AKS: full Kubernetes control for complex platforms.
2. Secrets in config → Key Vault/managed identity; Windows auth → Entra ID; file shares → Azure Files/Blob; MSMQ → Service Bus; scheduled tasks → Functions/Container Apps jobs; session state → Redis; logging → Application Insights.
3. Assess and plan; upgrade runtimes/frameworks for Java, .NET and C++; apply Azure migration tasks; fix builds and tests; scan and fix CVEs; generate containers, IaC and CI/CD; portfolio assessment via CLI.
4. Discovery and assessment of web apps and agentless ASP.NET migration at scale to App Service (plus the App Service migration assistant for single apps).
5. Incrementally routing functionality to new services behind a facade until the legacy system can be retired.
6. Generated changes can be wrong or incomplete; review, tests and security scans ensure correctness and safety.

</details>

## References

- [GitHub Copilot modernization](https://learn.microsoft.com/azure/developer/github-copilot-app-modernization/overview)
- [Copilot modernization for .NET](https://learn.microsoft.com/dotnet/core/porting/github-copilot-app-modernization/overview)
- [Java upgrade with Copilot](https://learn.microsoft.com/java/upgrade/overview)
- [App Service migration assistant](https://learn.microsoft.com/azure/app-service/app-service-migration-assistant)
- [Strangler fig pattern](https://learn.microsoft.com/azure/architecture/patterns/strangler-fig)
