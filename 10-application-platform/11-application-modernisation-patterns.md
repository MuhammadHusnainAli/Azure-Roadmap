# Application Modernisation Patterns

## Why it matters

Most organisations do not start on a blank page. They have monoliths, ageing .NET Framework and Java applications, mainframe-era batch jobs and hand-built integrations, and they want them to be cheaper, safer, faster to change, and ready for AI. Modernisation is a sequence of trade-offs: how much to change, in what order, and how to keep the business running meanwhile. Architects design the target and the path; consultants build the business case and plan; engineers do the work, increasingly with AI assistance.

## Key concepts

### The migration and modernisation strategies

| Strategy | Meaning | Example | Effort | Benefit |
|---|---|---|---|---|
| **Retire** | Switch it off | Unused reporting app | Lowest | Removes cost and risk |
| **Retain** | Leave as is for now | Stable app near end of life | None | Defers effort |
| **Rehost** ("lift and shift") | Move to Azure VMs with minimal change | IIS app on a VM | Low | Fast exit from datacentre; few cloud benefits |
| **Replatform** ("lift and reshape") | Move to managed services with small changes | App to App Service, database to SQL Managed Instance | Low to medium | Managed patching, scaling, HA |
| **Refactor** | Restructure code for the cloud without changing behaviour | Containerise, externalise state, add managed identity | Medium | Cloud-native operations |
| **Rearchitect** | Change the architecture | Split a monolith into services, event-driven design | High | Scalability, agility |
| **Rebuild / replace** | Rewrite, or adopt SaaS | Move CRM to Dynamics 365 or Salesforce | High (rebuild) / variable (replace) | Fit to current needs |

Most estates mix strategies per application. The Cloud Adoption Framework's **Migrate** and **Modernize** guidance and Azure Migrate's assessments support the decision.

### Choosing a strategy per application

| Question | Points toward |
|---|---|
| Is the business value high and the app changing often? | Refactor or rearchitect |
| Is there a hard deadline (datacentre exit, hardware end of life)? | Rehost first, modernise later |
| Is there a SaaS that meets the need? | Replace |
| Is it stable, low-change, and supported? | Replatform or retain |
| Is the technology unsupported (old .NET Framework, Java 8, Windows Server 2012)? | Replatform with upgrade, or refactor |
| Will AI features be added soon? | Modernise the data access and APIs the AI needs first |

### The strangler fig pattern

Replace a system incrementally by routing functionality to new components until the old system can be removed.

```text
Phase 1                           Phase 2                              Phase 3
Clients → Facade (APIM / Front Door / App Gateway)
            └─► Monolith (all)     ├─► New "orders" service             ├─► New services (all)
                                   └─► Monolith (everything else)       └─ Monolith retired
```

| Step | Detail |
|---|---|
| 1. Introduce a facade | API Management, Application Gateway, Front Door or a reverse proxy in front of the monolith; clients now call the facade |
| 2. Pick a seam | A bounded capability with clear inputs and outputs (for example notifications, catalogue, pricing) |
| 3. Build the new component | On Container Apps, Functions or App Service; own its data where possible |
| 4. Route traffic | Facade routes that path to the new component; use feature flags (file 07) for gradual rollout |
| 5. Synchronise data | During transition, keep data in sync (change data capture, events, anti-corruption layer) |
| 6. Remove the old code path | Once stable |
| 7. Repeat | Until the monolith is gone or small enough to keep |

Related patterns: **anti-corruption layer** (translate between old and new models), **backends for frontends**, **sidecar** and **ambassador** for cross-cutting concerns, **event-driven architecture** to decouple, **CQRS** where reads and writes diverge.

### The twelve-factor app

A checklist for cloud-ready services:

| Factor | Azure practice |
|---|---|
| 1. Codebase | One repository per deployable, many deploys |
| 2. Dependencies | Declared and isolated (requirements, lock files, containers) |
| 3. Config | Environment and App Configuration, not code |
| 4. Backing services | Databases, caches, queues as attached resources by URL and identity |
| 5. Build, release, run | CI/CD pipelines, immutable artefacts, slots or revisions |
| 6. Processes | Stateless; state in managed services |
| 7. Port binding | Self-contained HTTP servers in containers |
| 8. Concurrency | Scale out by process (autoscale, KEDA) |
| 9. Disposability | Fast start, graceful shutdown on `SIGTERM` |
| 10. Dev/prod parity | Same services and IaC in every environment |
| 11. Logs | Stdout to Azure Monitor / OpenTelemetry |
| 12. Admin processes | One-off jobs (Container Apps jobs, Functions) |

Modern additions often cited: API first, telemetry, and security through identity rather than secrets.

### AI-assisted modernisation

AI now does a meaningful share of modernisation work:

| Tool or capability | What it helps with |
|---|---|
| **GitHub Copilot app modernization** (for .NET and Java) | Assess an application, upgrade framework versions (for example .NET Framework → modern .NET, older Java → current LTS), fix breaking changes, apply Azure migration changes (managed identity, Key Vault, storage SDKs), containerise and generate deployment files |
| **GitHub Copilot agent mode / coding agent** | Iterative refactoring, tests, documentation of legacy code |
| **Azure Migrate** | Discovery, dependency mapping, application and code assessments, business case, and integration with Copilot-driven modernisation (check current features) |
| **Azure Copilot** | Questions about the target platform, generating infrastructure and troubleshooting |
| Code and schema conversion assistants | Converting procedural database code (for example Oracle PL/SQL to PostgreSQL) with review |

AI speeds up mechanical changes; humans still own architecture decisions, test strategy and validation. Build a **test safety net** before large AI-assisted refactors.

### Modernising for AI readiness

Applications become usable by agents when they expose clean, secure capabilities:

| Step | Outcome |
|---|---|
| Expose business capabilities as **APIs** behind API Management | Agents and apps call the same contracts |
| Publish them as **MCP servers** or OpenAPI tools | Agents can discover and use them |
| Move data to managed databases with **Fabric mirroring** or events | Data available for RAG and analytics |
| Adopt **Entra identities** end to end | On-behalf-of flows let agents act with user permissions |
| Add **telemetry** (OpenTelemetry) | Agent actions are observable and auditable |

## How it works on Azure

### A worked example

A 12-year-old ASP.NET MVC (.NET Framework 4.8) order system on Windows Server VMs with SQL Server 2014, nightly batch jobs and a shared file share.

| Phase | Actions | Azure targets |
|---|---|---|
| 0. Assess | Azure Migrate discovery and dependency mapping; Copilot app modernization assessment of the code | – |
| 1. Replatform quickly | Database to SQL Managed Instance (MI link); file share to Azure Files; app to App Service (or managed instance on App Service if it needs OS dependencies); batch jobs to Functions or Container Apps jobs | SQL MI, Azure Files, App Service |
| 2. Upgrade and refactor | Upgrade to modern .NET with Copilot assistance; managed identity everywhere; App Configuration and Key Vault; containerise | Container Apps or App Service (Linux) |
| 3. Strangle | Put API Management in front; extract "order status" and "notifications" as services; publish order status as an MCP tool for a customer service agent | APIM, Container Apps, Functions |
| 4. Optimise | Autoscale, reservations, Fabric mirroring of the database for analytics and RAG | – |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Speed vs benefit | Rehost when deadlines dominate, but schedule the follow-up; otherwise replatform as the default first step |
| Microservices | Split only along clear business boundaries and team ownership; a "distributed monolith" is worse than a monolith |
| Big bang vs incremental | Incremental (strangler fig) for anything business-critical |
| Data | Decide early which service owns which data; shared databases block independent change |
| Host choice | App Service for web apps; Container Apps for services and jobs; Functions for event-driven pieces; AKS only with a platform team and clear need |
| AI assistance | Use it for upgrades and repetitive refactoring with tests and reviews; not as a replacement for design |

## Hands-on

**Modernisation plan exercise.** Choose an application you know (or the worked example above) and produce a one-page plan:

1. Inventory: components, technologies and versions, data stores, integrations, users, pain points.
2. Strategy per component (rehost, replatform, refactor, rearchitect, replace, retire, retain) with a one-line reason.
3. Target Azure services per component.
4. A strangler fig sequence: facade, first seam, order of extraction.
5. Risks and mitigations (data sync, performance, skills).
6. How AI features will use the modernised application (APIs, MCP tools, data).

If you have a sample .NET Framework or Java 8 project, try GitHub Copilot app modernization in Visual Studio or Visual Studio Code: run an assessment and an upgrade on a branch, and review every change it makes.

## Security, cost and reliability notes

- **Security:** modernisation is the moment to remove stored credentials, adopt managed identities, enforce TLS, and add WAF and private networking.
- **Security:** during strangler fig transitions, both old and new paths must be secured; facades must not open new holes.
- **Cost:** run old and new in parallel only as long as necessary; measure cost per transaction before and after.
- **Reliability:** each extracted service adds network calls and failure modes; add retries, timeouts, circuit breakers and observability.
- **Reliability:** keep rollback routes at the facade until each step is proven.

## Common mistakes

- Rehosting and never modernising, ending up with a costlier datacentre in the cloud.
- Rewriting everything from scratch, delivering nothing for two years.
- Splitting into many microservices that share one database.
- Modernising without tests, then being unable to verify behaviour.
- Accepting AI-generated refactors without review.
- Ignoring data migration and synchronisation until late.

## Check yourself

1. Name the seven common strategies and give an example of each.
2. Describe the strangler fig pattern in five steps.
3. What is an anti-corruption layer?
4. Which twelve-factor principles matter most for running on Container Apps?
5. What can GitHub Copilot app modernization do?
6. What makes an application "AI-ready"?
7. When is rehosting the right first step?

<details>
<summary>Answers</summary>

1. Retire (unused app switched off), retain (stable app left alone), rehost (VM lift and shift), replatform (app to App Service, DB to MI), refactor (containerise, externalise state), rearchitect (monolith to services), rebuild or replace (rewrite, or adopt SaaS).
2. Put a facade in front of the old system; choose a seam; build the new component; route that traffic to it gradually; remove the old code path and repeat.
3. A translation layer between new and legacy components so the new design is not polluted by the old system's models and interfaces.
4. Config in the environment, stateless processes, disposability (fast start, graceful shutdown), logs to stdout, port binding, concurrency by scaling out, backing services as attached resources.
5. Assess .NET and Java applications, upgrade framework and runtime versions, fix breaking changes, apply Azure-specific changes such as managed identity and SDK updates, and help containerise and deploy.
6. Clean, secured APIs (and MCP or OpenAPI tools) for its capabilities, data accessible for retrieval and analytics, end-to-end Entra identity for delegated access, and good telemetry.
7. When a hard deadline such as a datacentre exit or hardware end of life dominates, with modernisation planned afterwards.

</details>

## References

- [Cloud Adoption Framework: Migrate](https://learn.microsoft.com/azure/cloud-adoption-framework/migrate/)
- [Cloud Adoption Framework: Modernize](https://learn.microsoft.com/azure/cloud-adoption-framework/modernize/)
- [Strangler fig pattern](https://learn.microsoft.com/azure/architecture/patterns/strangler-fig)
- [Anti-corruption layer pattern](https://learn.microsoft.com/azure/architecture/patterns/anti-corruption-layer)
- [Modern web app pattern](https://learn.microsoft.com/azure/architecture/web-apps/guides/enterprise-app-patterns/modern-web-app/dotnet/guidance)
- [GitHub Copilot app modernization](https://learn.microsoft.com/azure/developer/github-copilot-app-modernization/overview)
- [Azure Migrate application assessments](https://learn.microsoft.com/azure/migrate/concepts-azure-webapps-assessment-calculation)
- [The Twelve-Factor App](https://12factor.net/)
