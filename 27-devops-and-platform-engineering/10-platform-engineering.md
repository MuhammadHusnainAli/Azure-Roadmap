# Platform Engineering

## Why it matters

As organisations scale cloud adoption, every application team ends up solving the same problems: CI/CD, landing zone onboarding, networking, identity, observability, security scanning, cost controls, and now AI gateways and agent governance. Without a platform, each team reinvents them inconsistently, cognitive load explodes and security drifts. **Platform engineering** applies a **product mindset** to internal tooling: a dedicated team builds an **internal developer platform (IDP)** with **self-service** capabilities and **paved roads** that make the secure, compliant, cost-effective way the easiest way. Microsoft notes that small platform teams (fewer than 20 people) can support thousands of developers when the platform is built incrementally and treated as a product.

## Key concepts

### Principles

| Principle | Meaning |
|---|---|
| **Product mindset** | Developers are customers; discover needs, prioritise, measure adoption and satisfaction |
| **Self-service** | Developers provision what they need through APIs, templates, pipelines or a portal without tickets |
| **Paved roads / golden paths** | Opinionated, supported paths to production with security and compliance built in |
| **Reduce cognitive load** | Abstract infrastructure complexity without hiding it entirely |
| **Incremental** | Start with the biggest pain point; avoid a big-bang platform |
| **Everything as code** | Templates, policies, pipelines and docs versioned and reviewed |
| **Optional but attractive** | Teams can leave the paved road with justification; the road should be better |

### Capabilities of an internal developer platform

| Capability | Azure/GitHub building blocks |
|---|---|
| **Application templates** | Template repositories, `azd` templates, dev containers |
| **Infrastructure self-service** | AVM-based Bicep/Terraform modules, deployment stacks, template specs, subscription vending ([Topic 03](../03-governance-and-landing-zones/README.md)) |
| **CI/CD** | Reusable workflows, Azure Pipelines templates, Managed DevOps Pools |
| **Security and compliance** | Azure Policy, GitHub rulesets, Advanced Security, Defender for Cloud, OIDC |
| **Observability** | Default Application Insights, diagnostic settings, dashboards and alert templates (Topic 26) |
| **AI platform** | Shared Foundry resources, AI gateway in APIM, approved models and MCP registry (API Center), evaluation templates |
| **Developer portal** | Backstage (open source), Port or a custom portal; GitHub as the de facto portal for many teams |
| **Service catalogue** | Ownership, dependencies, scorecards (API Center for APIs; Backstage catalogue) |
| **Cost visibility** | Tags, budgets, showback (Topic 29) |

### Measuring developer experience

| Framework / metric | Measures |
|---|---|
| **DORA** | Deployment frequency, lead time for changes, change failure rate, time to restore (plus reliability) |
| **SPACE** | Satisfaction, Performance, Activity, Communication, Efficiency |
| **DevEx** | Feedback loops, cognitive load, flow state |
| **Platform metrics** | Adoption of golden paths, time to first deployment for new services, onboarding time, ticket volume, % of resources compliant |

Combine system data (pipelines, Git) with developer surveys; don't use metrics to rank individuals.

### Team topology

Platform team as an **enabling, product-oriented** team serving stream-aligned (application) teams, with clear APIs and documentation, and a feedback loop (office hours, champions, roadmap).

## How it works on Azure

Reference platform on Azure and GitHub:

```text
Developer portal (Backstage / GitHub) ── templates ──► new repo with app code, devcontainer, workflows, IaC
                                     ── self-service ──► subscription vending / environment workflows
Platform repos: reusable workflows, AVM modules, policy-as-code, rulesets, dashboards, alert baselines
Landing zones: management groups, policies, hub networking, central logging, Defender, AI gateway
Feedback: DORA metrics from GitHub/Azure DevOps, surveys, platform adoption dashboard
```

Example golden path for "containerised API":

| Layer | Default |
|---|---|
| Repo | Template with ASP.NET/FastAPI skeleton, tests, `devcontainer.json`, `CODEOWNERS` |
| Infra | Bicep module: Container Apps environment (internal), managed identity, Key Vault, App Insights |
| Pipeline | Reusable build/scan/sign/deploy workflow with OIDC and environments |
| Security | CodeQL, secret scanning, Defender for Containers, Notation signing |
| Observability | OpenTelemetry distro, standard dashboard and alerts |
| Docs | Runbook template, SLO template |

Measuring DORA from GitHub (sketch): deployments via the Deployments API per environment, lead time from first commit to production deployment, change failure rate from deployments followed by incidents or rollbacks.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Build vs buy portal** | Start with GitHub templates and workflows; add Backstage or a commercial portal when catalogue and discovery needs grow. |
| **Abstraction level** | Thin abstractions over Azure services (modules, templates) rather than a proprietary platform that hides Azure entirely. |
| **Mandate vs attract** | Mandate security baselines via policy; attract adoption of golden paths through quality. |
| **Platform scope** | Prioritise by developer pain and risk; avoid building everything. |
| **Versioning** | Semantic versions for modules and workflows; deprecation policy and migration support. |
| **Retiring services** | Track Azure lifecycle changes (for example Dev Box and Deployment Environments) in the platform roadmap (file 09). |

## Hands-on

1. Interview three development teams; list their top five pain points.
2. Build one golden path (template repo + reusable workflow + Bicep module) for the top pain point.
3. Measure time to first production deployment for a new service before and after.
4. Collect DORA metrics for two teams from your CI/CD system.
5. Publish a platform roadmap and a deprecation policy for modules.

## Security, cost and reliability notes

- **Secure by default**: golden paths encode identity, network, scanning and logging defaults.
- **Blast radius**: shared modules and workflows are powerful; protect them with review, versioning and testing.
- **Cost**: platform team cost is justified by reduced duplication; show the ROI with metrics.
- **Reliability**: shared CI infrastructure and templates are critical services; monitor them.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building a platform nobody asked for | Product discovery with developers |
| Big-bang platform programme | Incremental, pain-driven |
| Mandating everything | Policies for baselines; quality for adoption |
| No metrics | DORA, DevEx surveys, adoption |
| Breaking changes to shared templates | Versioning and deprecation policy |

## Check yourself

1. What is platform engineering, and how does it differ from traditional ops?
2. List five capabilities of an internal developer platform and Azure building blocks for each.
3. What is a golden path?
4. Name the DORA metrics.
5. How should platform adoption be encouraged?
6. Why version shared modules and workflows?

<details>
<summary>Answers</summary>

1. A product-minded practice building self-service internal platforms that reduce developer cognitive load and embed security and compliance, rather than ticket-driven operations.
2. Any five: templates (template repos, azd), infra self-service (AVM, stacks, subscription vending), CI/CD (reusable workflows, Managed DevOps Pools), security (Policy, rulesets, Defender), observability (App Insights defaults), AI platform (Foundry, AI gateway, API Center), portal (Backstage), cost visibility (tags, budgets).
3. The supported, opinionated, easiest route to production for a workload type, with built-in defaults.
4. Deployment frequency, lead time for changes, change failure rate, time to restore service.
5. Make golden paths high quality and convenient; mandate only baselines via policy; gather feedback.
6. To let teams adopt changes safely, avoid breaking consumers and support deprecation.

</details>

## References

- [What is platform engineering?](https://learn.microsoft.com/platform-engineering/what-is-platform-engineering)
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Subscription vending](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/subscription-vending)
- [DORA research](https://dora.dev/)
