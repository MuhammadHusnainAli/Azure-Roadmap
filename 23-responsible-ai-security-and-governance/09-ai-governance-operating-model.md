# AI Governance Operating Model

## Why it matters

Principles, guardrails and regulations only matter if the organisation has a way to apply them to every AI use case consistently, without blocking every team for months. Without an operating model you get one of two failures: **shadow AI** (teams build or buy AI with no oversight, discovered only after an incident) or **governance theatre** (a committee that reviews slide decks while nothing is enforced in the platform). A working AI governance operating model defines who decides, how use cases enter and progress, what evidence is required at each gate by risk tier, how models and agents are inventoried, and how policies are enforced automatically in Azure. Microsoft's Cloud Adoption Framework (CAF) describes it as four steps: assess risks, document policies, enforce policies, monitor risks.

## Key concepts

### CAF's governance process

| Step | Activities |
|---|---|
| **1. Assess AI organisational risks** | Understand each workload; assess against responsible AI principles; identify security, operational, ethical, third-party and integration risks |
| **2. Document AI governance policies** | Model selection and onboarding; third-party tools and data; maintenance and monitoring; regulatory compliance; user conduct |
| **3. Enforce AI governance policies** | Automate with Azure Policy, Purview, Defender; consistent enforcement across platforms |
| **4. Monitor AI organisational risks** | Continuous monitoring, audits, risk reviews, policy updates |

### Structures and roles

| Body or role | Responsibility |
|---|---|
| **Executive sponsor / AI steering group** | Strategy, risk appetite, funding, final escalations |
| **AI governance board (or Responsible AI council)** | Policies, standards, risk tiering, approval of high-risk use cases, exceptions |
| **AI Centre of Excellence (CoE)** | Platform, reusable patterns, guidance, training, intake triage |
| **Responsible AI champions** | Embedded in business units; first-line guidance and assessments |
| **Use-case owner** | Accountable for value, risk acceptance and operation of a specific AI system |
| **Security, privacy, legal, compliance** | Specialist review and sign-off by tier |
| **Internal audit** | Independent assurance (third line) |

Align with the existing **three lines** model: business owners (first line), risk and compliance functions (second), internal audit (third). CAF recommends assigning agent governance to the same leaders responsible for cloud governance, security and compliance.

### Risk tiering

| Tier | Examples | Process |
|---|---|---|
| **Low** | Internal drafting assistant on public data; Microsoft 365 Copilot for general productivity | Self-assessment, standard guardrails, register |
| **Medium** | Customer-facing Q&A with RAG; internal agent with read tools | CoE review, evaluation evidence, privacy check |
| **High** | Decisions affecting people (credit, HR), agents with write tools on critical systems, EU AI Act high-risk | Full impact assessment, board approval, red teaming, human oversight design, legal review |
| **Prohibited** | Uses banned by law or policy | Rejected at intake |

### Intake and lifecycle gates

```text
Idea ─► Intake form ─► Tiering ─► Design review ─► Pre-production gate ─► Launch ─► Periodic review ─► Retire
          │                │            │                   │                         │
      inventory entry   risk tier   impact assessment   evaluation + red team    KPIs, incidents,
                                    threat model        + sign-offs              re-assessment
```

The intake form captures: purpose, owner, users, data categories, model(s), tools and actions, autonomy level, jurisdictions, vendor/build, expected value. It creates the **inventory entry** immediately.

### Model and agent inventory

| Field | Why |
|---|---|
| ID, name, owner, business unit | Accountability |
| Type (model deployment, RAG app, agent, Copilot extension, SaaS AI) | Scope of controls |
| Risk tier and regulatory classification | Gate requirements |
| Models and versions, tools, data sources, identity | Change impact and incident response |
| Guardrail, evaluation results, red team results | Evidence |
| Status, review dates, retirement date | Lifecycle |

Sources: Foundry Control Plane and Agent 365 registries (file 10), Entra Agent ID, Azure Resource Graph for AI resources, Defender for Cloud AI inventory, Purview DSPM app discovery, Copilot Studio inventory in the Power Platform admin centre. CAF's guidance: if Agent 365 is available, use its registry; otherwise use Entra Agent ID as the authoritative source for agent identities and ownership.

### Policy enforcement in the platform

| Policy | Enforcement |
|---|---|
| Approved models only | Azure Policy on allowed model deployments; Foundry Control Plane model policies |
| No public network access, no local auth | Azure Policy (deny/modify) on Foundry, Search, Storage |
| Guardrail required | Policy/IaC requiring custom `raiPolicyName`; review agent guardrails |
| Allowed regions and deployment types | Azure Policy |
| Defender plans on | Azure Policy `DeployIfNotExists` |
| Tagging (owner, tier, cost centre) | Azure Policy |
| Data protection | Purview labels, DLP, DSPM policies |
| Approved tools | API Center / private MCP registry; APIM gateway |

Enforce at **management group** scope through the AI landing zone ([Topic 03 file 04](../03-governance-and-landing-zones/04-ai-landing-zones.md)).

## How it works on Azure

Example: Azure Policy assignment denying public network access on Foundry resources (built-in policies exist for Azure AI services; search for "Azure AI Services resources should restrict network access" and "should have key access disabled"):

```bash
az policy assignment create \
  --name deny-ai-public-network \
  --scope /providers/Microsoft.Management/managementGroups/mg-landing-zones \
  --policy "<built-in policy definition ID>" \
  --params '{ "effect": { "value": "Deny" } }'
```

Example Resource Graph query listing AI resources and owners for the inventory:

```kusto
resources
| where type in~ ("microsoft.cognitiveservices/accounts", "microsoft.machinelearningservices/workspaces", "microsoft.search/searchservices")
| project name, type, kind, location, resourceGroup, subscriptionId, owner = tostring(tags["owner"]), tier = tostring(tags["ai-risk-tier"])
```

Governance dashboards combine Resource Graph, Defender for Cloud secure score and AI recommendations, Foundry Control Plane compliance, Purview DSPM reports and Compliance Manager scores.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Centralised vs federated** | Central policy and platform; federated delivery with champions. Central ownership improves consistency but needs clear decision rights. |
| **Proportionality** | Low-risk use cases should take days, not months; otherwise teams bypass the process. |
| **Paved road** | Make the governed platform (approved models, guardrails, templates) the easiest path. |
| **Automate evidence** | Collect evaluation results, policy compliance and inventory automatically from CI/CD and Azure. |
| **Exceptions** | Time-bound, owner-approved, recorded in the register. |
| **Buy vs build** | Apply the same intake to SaaS AI features and third-party agents. |

## Hands-on

1. Draft a one-page AI policy covering acceptable use, model selection, data, human oversight and incident reporting.
2. Design an intake form and risk-tiering questionnaire (ten questions).
3. Define gate evidence per tier in a table.
4. Assign Azure Policies for no public access, no local auth and required tags on a test management group.
5. Build the Resource Graph inventory query into a workbook alongside Foundry Control Plane and Defender views.
6. Run one real use case through the process and time it.

## Security, cost and reliability notes

- **Shadow AI** is a security and cost risk: discover via Defender, Purview DSPM and network monitoring.
- **Governance cost**: reviews consume expert time; tiering keeps it focused.
- **Platform drift**: re-run compliance checks continuously, not only at launch.

## Common mistakes

| Mistake | Fix |
|---|---|
| One heavyweight process for all use cases | Risk tiers with proportional gates |
| Inventory in a spreadsheet updated quarterly | Automated discovery plus registry |
| Policies with no technical enforcement | Azure Policy, Purview, guardrail standards |
| Governance only for custom builds | Include SaaS AI, Copilot extensions and agents |
| No retirement process | Review dates and decommissioning steps |

## Check yourself

1. What are CAF's four AI governance steps?
2. Name four roles or bodies in an AI governance operating model.
3. How should process differ by risk tier?
4. What fields belong in an AI inventory entry?
5. Which Azure tools provide inventory signals?
6. Give four policies you can enforce with Azure Policy for AI resources.

<details>
<summary>Answers</summary>

1. Assess AI organisational risks, document governance policies, enforce policies, monitor risks.
2. Any four: executive sponsor/steering group, governance board/RAI council, CoE, RAI champions, use-case owner, security/privacy/legal reviewers, internal audit.
3. Low: self-assessment and registration; medium: CoE review and evaluation evidence; high: full impact assessment, board approval, red teaming, legal review; prohibited: rejected.
4. ID, owner, type, risk tier, models/versions, tools, data sources, identity, guardrail, evaluation and red team evidence, status and review dates.
5. Foundry Control Plane, Agent 365, Entra Agent ID, Azure Resource Graph, Defender for Cloud AI inventory, Purview DSPM, Power Platform admin centre.
6. Any four: allowed models, no public network access, no local auth, allowed regions/deployment types, Defender plans enabled, required tags.

</details>

## References

- [Govern AI (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/govern)
- [Manage AI (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/manage)
- [Govern and secure AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization)
- [Foundry Control Plane overview](https://learn.microsoft.com/azure/foundry/control-plane/overview)
