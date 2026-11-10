# Cloud Adoption Framework

## Why it matters

The Microsoft Cloud Adoption Framework (CAF) is the shared vocabulary between Microsoft, partners and customers for how to adopt Azure. Clients and Microsoft account teams use its terms, assessments reference it, and partner funding programmes are organised around it. A consultant who can place a client's situation in CAF and name the next decision is immediately useful. An architect uses it to make sure the technical design sits on a sound organisational footing.

CAF is guidance for **decision makers** about the whole organisation's adoption. The Well-Architected Framework (topic 33) is guidance for **workload teams** about a single workload. You need both, and they are not interchangeable.

## Key concepts

### The seven phases

| # | Phase | Key decision | Main outputs |
|---|---|---|---|
| 1 | **Strategy** | What should our Azure adoption look like? | Motivations, business outcomes, business case, strategy document |
| 2 | **Plan** | How will we prepare for adoption? | Digital estate inventory, prioritised workloads, skills plan, adoption plan |
| 3 | **Ready** | How will we build our Azure landing zone? | Platform landing zone, workload landing zone process |
| 4 | **Adopt** | How will we migrate, modernise and build workloads? | Migrated, modernised and new cloud-native workloads |
| 5 | **Govern** | How will we control our environment? | Governance policies, risk register, enforcement and monitoring |
| 6 | **Secure** | How will we protect our environment? | Security posture, operations and incident response aligned to Zero Trust |
| 7 | **Manage** | How will we operate and optimise over time? | Operations baseline, monitoring, resilience, cost management |

Strategy to Adopt flow roughly in sequence. Govern, Secure and Manage run continuously alongside.

CAF is iterative. Organisations do not finish Strategy and never revisit it; each new wave of workloads, and now each new technology such as AI, sends them around again.

### Strategy

- **Motivations:** why move? Common drivers are data centre exit, cost, agility, resilience, innovation and, increasingly, readiness for AI.
- **Business outcomes:** measurable results, not "be in the cloud".
- **Business case:** financial model comparing current and future state.
- **Sponsorship and alignment:** an executive owner and agreed priorities.

A consultant's early job is often to turn a vague motivation into a measurable outcome.

### Plan

- **Digital estate rationalisation:** inventory workloads and assign each a treatment.
- **Skills and organisation:** who does what; where the gaps are.
- **Adoption plan:** prioritised backlog with owners and dates.

The treatments, often called the Rs:

| Treatment | Meaning |
|---|---|
| Retire | Switch it off |
| Rehost | Move as-is to IaaS |
| Replatform | Move with minor changes to PaaS |
| Refactor | Change code to suit the cloud |
| Rearchitect | Redesign significantly |
| Rebuild | Write again cloud-native |
| Replace | Adopt SaaS instead |
| Retain | Leave where it is, for now |

### Ready

Build the environment workloads will land in: the **Azure landing zone**. Covered in files 02 to 04.

### Adopt

Three paths, usually mixed: **migrate**, **modernise**, and **build cloud-native**. Topic 30 covers migration.

### Govern

Governance in CAF is a cycle, not a product:

1. **Build a governance team.**
2. **Assess cloud risks:** regulatory, security, operations, cost, data, resource management, and AI.
3. **Document governance policies:** what must be true, in plain language.
4. **Enforce governance policies:** with Azure Policy, RBAC, IaC, and process.
5. **Monitor governance:** compliance reporting, review, improvement.

The governance policy ("AI resources must not be exposed to the public internet") is the requirement. The Azure Policy definition is one way of enforcing it.

### Secure

Aligns adoption with Zero Trust and the **CIA triad** (confidentiality, integrity, availability), across security posture, incident preparation and response, and the organisation's security operating model. Topic 25.

### Manage

Operating the estate: inventory and visibility, operational compliance, protection and recovery, platform and workload operations. Recent guidance frames this as **RAMP**: Ready, Administer, Monitor, Protect.

### Scenarios

Scenarios adapt CAF to particular technologies and needs. They build on the foundation above.

| Scenario | Purpose |
|---|---|
| **Azure adoption** | The foundational path |
| **Data platform** | Preparing data for AI and analytics |
| **AI adoption** | Adopting generative and non-generative AI workloads |
| **AI agents** | How and when to adopt AI agents |
| **Sovereignty** | Meeting sovereignty requirements |
| **Azure VMware Solution** | Adopting AVS |

### The AI adoption scenario

It re-runs the CAF phases with AI-specific decisions.

| Phase | AI-specific decisions |
|---|---|
| **AI Strategy** | Identify use cases and success metrics; choose a technology approach; define a data strategy; define a responsible AI strategy |
| **AI Plan** | Assess skills; prioritise use cases; run proofs of concept; plan responsible AI implementation |
| **AI Ready** | Governance boundaries, networking and reliability foundations for AI workloads (file 04) |
| **Govern AI** | Assess AI risks; document, enforce and monitor AI governance policies |
| **Secure AI** | Identify AI-specific threats; protect AI resources, data and access; detect and respond |
| **Manage AI** | Operations for models and agents: deployment, monitoring, cost, data, business continuity |

**The technology approach decision** is the one consultants are asked about most. CAF frames it as a spectrum:

| Approach | Product | You manage | Fits when |
|---|---|---|---|
| **SaaS** | Microsoft 365 Copilot, role-based Copilots | Licences, data hygiene, adoption | The need is general productivity over Microsoft 365 data |
| **Low-code PaaS** | Copilot Studio | Agent design, connectors, governance | Business teams need tailored agents quickly |
| **Pro-code PaaS** | Microsoft Foundry | Models, retrieval, orchestration, application | You need control over behaviour, data and integration |
| **IaaS** | GPU VMs, AKS | Everything, including the models | You must bring your own model or have constraints PaaS cannot meet |

Start as far up as meets the requirement. This mirrors the "highest abstraction" rule from topic 01.

### The AI agents scenario

Separate guidance for deciding **whether** a use case warrants an agent, and how to plan, govern, build and operate agents across the organisation. Its core message: not every problem needs an agent, single-agent designs should be the default, and identity, data and governance foundations must be in place before scaling.

### Operating models

How an organisation divides cloud responsibilities.

| Model | Description | Trade-off |
|---|---|---|
| **Decentralised** | Each workload team runs everything | Fast, inconsistent, duplicated effort |
| **Centralised** | A central IT team runs everything | Consistent, becomes a bottleneck |
| **Enterprise (platform + workload teams)** | A platform team provides landing zones and guardrails; workload teams have autonomy within them | The target for most organisations at scale |
| **Distributed** | A mix across business units, often after acquisitions | Realistic, complex to govern |

The enterprise model is what Azure landing zones are built for.

**Teams and functions** CAF describes: cloud strategy, cloud adoption, cloud governance, cloud platform, cloud operations, cloud security, and a **cloud centre of excellence (CCoE)** that sets standards and enables others. For AI, an **AI centre of excellence** plays the same enabling role (topic 34).

## How it works on Azure

### CAF tools you can use with clients

| Tool | Use |
|---|---|
| **Cloud Adoption Strategy Evaluator** | Assess strategy maturity |
| **Cloud Journey Tracker / SMART assessments** | Find where the client is and what to do next |
| **Strategy and plan templates** | Capture motivations, outcomes and the adoption plan |
| **Governance Benchmark** | Identify governance gaps |
| **Azure landing zone review** | Compare an existing environment with the reference |
| **AI readiness and technical assessments** | Gauge preparedness for AI adoption |
| **Azure Migrate** | Build the digital estate inventory and business case |

These live under *Assessments* on Microsoft Learn and in the CAF documentation.

### Mapping a client to CAF

A quick diagnostic you can run in a first meeting:

| If you hear… | They are probably in… | Next step |
|---|---|---|
| "We need to be doing something with AI" | Strategy | Define use cases and outcomes |
| "We have twenty ideas and no priority" | Plan | Score for value and feasibility; pick two for proof of concept |
| "Every team has built its own thing in its own subscription" | Ready and Govern, skipped | Establish a landing zone and baseline policies |
| "The pilot worked but security will not approve production" | Secure and Govern | Risk assessment, policy, network and identity design |
| "It is live but nobody knows what it costs or who supports it" | Manage | Operations baseline, cost allocation, ownership |

## Design decisions and trade-offs

- **How much foundation before the first workload?** Building a complete enterprise landing zone before delivering anything stalls momentum. Delivering workloads with no foundation creates debt. A common compromise: deploy the landing zone accelerator with defaults, land the first workload, and refine.
- **Start small and expand vs enterprise scale.** Smaller organisations can begin with a simpler structure, provided it follows the same design principles and can grow.
- **Central control vs team autonomy.** Guardrails enforced by policy let you grant autonomy safely. Approval boards for every change do not scale.
- **Framework as checklist vs framework as guide.** CAF is not a compliance standard. Use it to find the decisions you have not yet made, not to tick boxes.

## Hands-on

A consulting exercise. Take a real or imagined organisation and produce a one-page CAF summary:

1. **Strategy:** write three motivations and, for each, one measurable outcome.
2. **Plan:** list six workloads and assign each a treatment with a one-line reason.
3. **Ready:** state whether a landing zone exists and what is missing.
4. **Govern:** write five governance policies in plain language (for example "Production data must stay in the EU").
5. **AI:** choose two AI use cases and place each on the SaaS, low-code, pro-code, IaaS spectrum with justification.
6. **Operating model:** say which model the organisation uses today and which it should move to.

Then run one of the Microsoft assessments (for example the AI readiness assessment) and compare its recommendations with your page.

## Security, cost and reliability notes

- **Security:** Secure is a continuous phase. Bolting security on after Adopt is the pattern CAF exists to prevent.
- **Cost:** the business case in Strategy and the cost management discipline in Manage must connect. Many adoptions set a savings target and never measure it.
- **Reliability:** resilience requirements belong in Plan (per workload) and Ready (platform), not in a post-incident review.

## Common mistakes

- Treating CAF as a one-time project plan.
- Skipping Strategy and Plan because "we just need to migrate".
- Confusing CAF with the Well-Architected Framework.
- Building the landing zone and stopping, with no operating model to run it.
- Starting AI adoption with a technology choice instead of a use case.
- Assuming every AI use case needs custom development in Foundry.
- Writing governance policies nobody enforces or monitors.

## Check yourself

1. List the seven CAF phases in order.
2. How does CAF differ from the Well-Architected Framework?
3. What are the five steps of the Govern cycle?
4. Name four of the migration treatments.
5. On the AI technology spectrum, where do Microsoft 365 Copilot, Copilot Studio and Microsoft Foundry sit?
6. What is the enterprise operating model?
7. A client has a working AI pilot that security will not approve. Which phases need attention?

<details>
<summary>Answers</summary>

1. Strategy, Plan, Ready, Adopt, Govern, Secure, Manage.
2. CAF guides organisation-wide adoption decisions for leaders and platform teams. The Well-Architected Framework guides the design of an individual workload.
3. Build a governance team, assess cloud risks, document governance policies, enforce them, monitor them.
4. Any four of retire, rehost, replatform, refactor, rearchitect, rebuild, replace, retain.
5. SaaS, low-code PaaS and pro-code PaaS respectively.
6. A central platform team provides landing zones and guardrails, and workload teams operate autonomously within them.
7. Govern and Secure, with Ready if the pilot was built outside a landing zone.

</details>

## References

- [Microsoft Cloud Adoption Framework for Azure](https://learn.microsoft.com/azure/cloud-adoption-framework/overview)
- [Strategy](https://learn.microsoft.com/azure/cloud-adoption-framework/strategy/)
- [Plan](https://learn.microsoft.com/azure/cloud-adoption-framework/plan/)
- [Govern](https://learn.microsoft.com/azure/cloud-adoption-framework/govern/)
- [Secure](https://learn.microsoft.com/azure/cloud-adoption-framework/secure/)
- [Manage](https://learn.microsoft.com/azure/cloud-adoption-framework/manage/)
- [AI adoption](https://learn.microsoft.com/azure/cloud-adoption-framework/ai/strategy)
- [AI agents adoption](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/)
- [Microsoft assessments](https://learn.microsoft.com/assessments/)
