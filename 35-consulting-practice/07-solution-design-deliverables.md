# Solution Design Deliverables

## Why it matters

Designs live on in documents long after the consultants leave. A clear **high-level design (HLD)** aligns stakeholders and approvers; a precise **low-level design (LLD)** lets engineers build without guessing; a **decision log** preserves the reasoning; and a **handover pack** lets operations teams run the system. Poor deliverables cause rework, failed security reviews, inconsistent builds and fragile operations. This file defines what each deliverable should contain for Azure and AI solutions, and how to keep them current with documentation-as-code.

## Key concepts

### Deliverables

| Deliverable | Audience | Purpose |
|---|---|---|
| **High-level design (HLD)** | Sponsors, architects, security, approvers | Solution overview, key decisions, how requirements are met |
| **Low-level design (LLD)** | Engineers, platform, security teams | Detailed configuration to build and review |
| **Decision log / ADRs** | Everyone, future teams | Why decisions were made (Topic 33 file 08) |
| **Security design / threat model** | Security, compliance | Controls, data flows, threats and mitigations |
| **Operations handover pack** | Operations/SRE, support | Runbooks, monitoring, SLOs, support model |
| **Test strategy and results** | QA, approvers | Functional, performance, security, AI evaluation evidence |
| **Cost estimate and FinOps plan** | Finance, owners | Run costs, budgets, optimisation |
| **Responsible AI documentation** | RAI board, compliance | Impact assessment, evaluation, transparency, oversight |

### HLD contents

| Section | Content |
|---|---|
| Context and scope | Business objectives, scope, out of scope |
| Requirements summary | Key functional requirements and NFRs (availability, RTO/RPO, performance, security, compliance) |
| Architecture overview | C4 context and container diagrams, Azure deployment view |
| Key decisions | Summary of ADRs and trade-offs (WAF pillars) |
| Data | Data flows, classification, residency, retention |
| Identity and security | AuthN/AuthZ, network isolation, secrets, threat model summary |
| AI components | Models, agents, tools, knowledge, guardrails, evaluation approach |
| Integration | Systems and interfaces |
| Operations | Monitoring, DR, deployment approach |
| Costs | Estimate summary |
| Risks, assumptions, open issues | With owners |

### LLD contents

Resource inventory per environment (names per convention, SKUs, regions, zones), networking (address spaces, subnets, NSGs, routes, private endpoints, DNS), identity (managed identities, role assignments, app registrations), configuration (app settings, Key Vault secrets references, feature flags), AI configuration (model deployments and versions, quotas/PTUs, agent definitions, tool contracts, guardrail settings, indexes and schemas), CI/CD (pipelines, environments, approvals), monitoring (DCRs, alerts, dashboards), backup/DR settings, policies and exemptions. Ideally, the LLD **is largely the IaC** plus a readable summary.

### Handover pack

| Item | Content |
|---|---|
| System overview | Architecture, dependencies, owners |
| Runbooks | Deploy, rollback, scale, failover, restore, rotate secrets, common incidents |
| Monitoring | Dashboards, alerts and their meanings, SLOs |
| Access | Roles, PIM, break-glass |
| AI operations | Evaluation pipelines, model upgrade process, guardrail tuning, red teaming schedule |
| Support model | Contacts, escalation, Microsoft support plan, vendor contracts |
| Known issues and backlog | Technical debt, planned improvements |
| Training | Sessions delivered, materials |

### Documentation as code

Store designs in Git alongside IaC (Markdown, Mermaid/diagrams-as-code, ADRs), review via pull requests, publish to a wiki (Azure DevOps Wiki, GitHub Pages) and version with releases.

## How it works on Azure

Generating LLD evidence from Azure:

```kusto
// Resource inventory for the LLD appendix
resources
| where resourceGroup startswith "rg-claims-"
| project name, type, location, sku = tostring(sku.name), zones = tostring(zones), tags
| order by type asc
```

```bash
# Role assignments for the identity section
az role assignment list --all --query "[?contains(scope, 'rg-claims-prod')].{principal:principalName, role:roleDefinitionName, scope:scope}" -o table
```

HLD diagram set: C4 context, C4 container, Azure deployment (regions, zones, VNets, private endpoints), data flow with trust boundaries, request sequence for the main AI flow (Topic 33 file 08).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Depth** | Proportional to risk and team size; avoid 200-page documents nobody reads. |
| **IaC as source of truth** | Generate inventories from Azure/IaC rather than maintaining duplicate tables. |
| **Templates** | Standard templates speed delivery and reviews. |
| **Living documents** | Update with changes via PRs; date and version every document. |
| **Audience-first** | Separate executive summaries from technical detail. |

## Hands-on

1. Create HLD and LLD templates for an Azure AI solution.
2. Write an HLD for a sample RAG assistant with C4 and deployment diagrams.
3. Generate an LLD resource inventory with Resource Graph.
4. Write three runbooks (deploy/rollback, failover, model upgrade).
5. Assemble a handover pack checklist and review it with an operations peer.

## Security, cost and reliability notes

- **Design documents contain sensitive details** (network ranges, security controls); classify and restrict.
- **Security design** reviewed by the client's security team before build.
- **Operational readiness** reviewed before go-live; handover is a gate, not an afterthought.

## Common mistakes

| Mistake | Fix |
|---|---|
| HLD full of LLD detail (or vice versa) | Audience-appropriate documents |
| Documents that drift from reality | Docs as code, generated inventories |
| No rationale for decisions | ADRs |
| Handover as a final-day email | Planned handover with training and acceptance |
| Missing AI-specific sections | Models, agents, evaluation, guardrails, RAI docs |

## Check yourself

1. Which deliverables make up a solution design package?
2. What does an HLD contain?
3. What belongs in an LLD for an AI solution?
4. What's in an operations handover pack?
5. What is documentation as code?
6. How can Azure tools generate LLD evidence?

<details>
<summary>Answers</summary>

1. HLD, LLD, decision log/ADRs, security design/threat model, operations handover pack, test strategy/results, cost estimate/FinOps plan, responsible AI documentation.
2. Context and scope, requirements summary, architecture overview, key decisions, data, identity and security, AI components, integration, operations, costs, risks/assumptions.
3. Resource inventory, networking, identity, configuration, AI configuration (models, agents, tools, guardrails, indexes), CI/CD, monitoring, backup/DR, policies.
4. System overview, runbooks, monitoring, access, AI operations, support model, known issues, training.
5. Keeping designs in Git as Markdown/diagram code, reviewed via PRs and versioned with releases.
6. Resource Graph inventories, role assignment exports, IaC, policy compliance and monitoring configuration exports.

</details>

## References

- [Architecture decision records](https://learn.microsoft.com/azure/well-architected/architect-role/architecture-decision-record)
- [Azure architecture icons](https://learn.microsoft.com/azure/architecture/icons/)
- [Azure Resource Graph](https://learn.microsoft.com/azure/governance/resource-graph/overview)
- [Well-Architected operational excellence](https://learn.microsoft.com/azure/well-architected/operational-excellence/principles)
