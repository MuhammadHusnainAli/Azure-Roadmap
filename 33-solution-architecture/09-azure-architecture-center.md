# Azure Architecture Center

## Why it matters

Few architecture problems are truly new. Someone at Microsoft or in the community has usually designed a baseline for a secure web app, a hub-spoke network, a RAG chat app, a data lakehouse or a multi-region API. The **Azure Architecture Center** is Microsoft's library of reference architectures, solution ideas, design guides, patterns, technology decision guides and Well-Architected-aligned baselines, many with deployable reference implementations on GitHub. Knowing how to find, evaluate and adapt these accelerates design, reduces risk and gives clients confidence that a solution follows Microsoft-validated practice.

## Key concepts

### Content types

| Type | What it is | Use |
|---|---|---|
| **Reference architectures (baselines)** | Detailed, WAF-reviewed architectures, often with reference implementations | Starting point for production designs (for example App Service baseline, AKS baseline, Foundry chat baseline) |
| **Solution ideas** | Lighter-weight architecture sketches for scenarios | Inspiration and early options |
| **Example workloads** | Real-world scenario architectures | Adapting to similar contexts |
| **Design guides** | Topic deep-dives (multitenancy, microservices, data, AI, networking) | Understanding principles and trade-offs |
| **Cloud design patterns** | Named patterns (file 04) | Solving recurring problems |
| **Technology decision guides** | Decision trees and comparisons (file 10) | Choosing services |
| **Antipatterns** | Common mistakes (noisy neighbour, chatty I/O, retry storm) | Reviews and troubleshooting |
| **Industry pages** | Finance, retail, healthcare, manufacturing scenarios | Industry context (Topic 32) |

### Evaluating a reference architecture

| Question | Why |
|---|---|
| Is it a baseline (production-ready) or a solution idea (sketch)? | Depth and validation differ |
| When was it last updated? | Services and recommendations change |
| Which WAF trade-offs does it make? | Fit to your priorities |
| Does it fit your constraints (regions, networking, landing zone, budget)? | Adaptation needed |
| Is there a reference implementation? | Faster, tested starting point |
| Which components are Preview or retiring? | Lifecycle risk |

### Adapting rather than copying

Start from the baseline, then apply your requirements: remove what you don't need (for example multi-region for an internal app), add what's missing (landing zone integration, your identity model, compliance controls), and record deviations in ADRs (file 08).

### Notable baselines (examples)

| Area | Baseline |
|---|---|
| Web | App Service baseline (zone-redundant, private), Container Apps microservices |
| Containers | AKS baseline cluster, AKS mission-critical |
| Networking | Hub-spoke, Virtual WAN, Azure Firewall designs |
| AI | **Baseline Microsoft Foundry chat** reference architecture, AI landing zone guidance, GenAI gateway, RAG solution design guide (Topic 34) |
| Data | Modern analytics with Fabric/Databricks, real-time analytics |
| Identity | Entra ID integration patterns |
| Landing zones | Azure landing zone accelerators (Topic 03) |

## How it works on Azure

Using a baseline with its reference implementation:

1. Find the baseline in the Architecture Center (browse by category or search).
2. Read the architecture, WAF considerations and trade-offs.
3. Clone the GitHub reference implementation; deploy to a sandbox subscription.
4. Validate behaviour (failover, private networking, monitoring).
5. Fork into your IaC repo; adapt to your landing zone (policy, networking, naming, tags).
6. Record deviations as ADRs; run a WAF review (file 02).

Architecture Center browse filters: product, category (AI + ML, containers, databases, hybrid, networking, security, web), and content type (reference architecture, solution idea, example workload).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Baseline vs custom** | Prefer a baseline and adapt; design from scratch only for novel requirements. |
| **Reference implementations** | Great accelerators; review code quality and maintenance before production. |
| **Currency** | Check update dates; cross-check with service docs for recent changes. |
| **Simplification** | Baselines are often more robust than needed; scale down deliberately. |
| **Client communication** | Cite Architecture Center sources to show alignment with Microsoft guidance. |

## Hands-on

1. Find three baselines relevant to your work and summarise their key decisions.
2. Deploy one reference implementation to a sandbox; inspect resources and networking.
3. Adapt it to a hypothetical landing zone; list deviations as ADRs.
4. Read two antipattern articles and check a system you know for them.
5. Build a personal index of Architecture Center resources by topic.

## Security, cost and reliability notes

- **Reference implementations** may include sample secrets or permissive settings for demos; harden before production.
- **Cost**: baselines often include premium SKUs and multiple zones; estimate before deploying.
- **Responsibility**: Microsoft guidance doesn't replace your validation and testing.

## Common mistakes

| Mistake | Fix |
|---|---|
| Copying a baseline without adapting | Apply your requirements and landing zone |
| Using outdated articles | Check update dates and service docs |
| Treating solution ideas as production designs | Use baselines for production |
| Deploying sample code directly to production | Review and harden |
| Ignoring antipatterns | Use them in reviews |

## Check yourself

1. Name the Architecture Center content types.
2. How do baselines differ from solution ideas?
3. What should you check when evaluating a reference architecture?
4. Describe the process for using a reference implementation.
5. Name three notable baselines.
6. Why record deviations from baselines?

<details>
<summary>Answers</summary>

1. Reference architectures/baselines, solution ideas, example workloads, design guides, patterns, technology decision guides, antipatterns, industry pages.
2. Baselines are detailed, WAF-reviewed and often have reference implementations; solution ideas are lighter sketches.
3. Type and depth, update date, WAF trade-offs, fit to constraints, reference implementation availability, Preview/retiring components.
4. Read, deploy to sandbox, validate, fork into IaC, adapt to landing zone, record deviations, review.
5. Any three: App Service baseline, AKS baseline, mission-critical, hub-spoke, baseline Foundry chat, GenAI gateway, modern analytics.
6. So reviewers and future teams understand why the design differs from Microsoft guidance.

</details>

## References

- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
- [Browse architectures](https://learn.microsoft.com/azure/architecture/browse/)
- [Architecture guides](https://learn.microsoft.com/azure/architecture/guide/)
- [Cloud design patterns](https://learn.microsoft.com/azure/architecture/patterns/)
