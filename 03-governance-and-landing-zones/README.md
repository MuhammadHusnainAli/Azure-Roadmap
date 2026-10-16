# 03 · Governance and Landing Zones

> Tracks: E ○ A ● C ● · Part A · Foundations

Governance is how an organisation keeps hundreds of people deploying to Azure without chaos: where things go, what is allowed, what it is called, and how you prove it. A landing zone is that governance made concrete, as a ready-to-use environment. This topic is where architects and consultants spend much of their early time on any serious engagement, and it is what makes an AI platform safe to scale beyond one team.

## What you will be able to do

- Walk a client through the Cloud Adoption Framework and place their current state in it.
- Design a management group and subscription structure and justify it.
- Explain platform and workload landing zones and the design areas behind them.
- Place AI workloads correctly in a landing zone and list the guardrails they need.
- Write, assign and remediate Azure Policy, including policies for AI services.
- Define a naming and tagging standard and enforce it.
- Interrogate an estate with Azure Resource Graph.
- Discuss compliance, residency and sovereignty accurately.

## Prerequisites

[Topic 01](../01-cloud-and-azure-fundamentals/README.md) (resource hierarchy, ARM) and [Topic 02](../02-identity-and-access/README.md) (RBAC, PIM).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Cloud Adoption Framework](01-cloud-adoption-framework.md) | 45 min | – |
| 02 | [Management Group and Subscription Design](02-management-group-and-subscription-design.md) | 50 min | CLI |
| 03 | [Azure Landing Zones](03-azure-landing-zones.md) | 60 min | Review |
| 04 | [AI Landing Zones](04-ai-landing-zones.md) | 60 min | Design |
| 05 | [Azure Policy](05-azure-policy.md) | 75 min | CLI, Bicep |
| 06 | [Naming, Tagging and Resource Organisation](06-naming-tagging-and-resource-organisation.md) | 40 min | CLI |
| 07 | [Azure Resource Graph](07-azure-resource-graph.md) | 45 min | KQL |
| 08 | [Regulatory Compliance](08-regulatory-compliance.md) | 45 min | Portal |
| 09 | [Azure Lighthouse and Managed Applications](09-azure-lighthouse-and-managed-applications.md) | 35 min | – |
| 10 | [Sovereign and Specialised Clouds](10-sovereign-and-specialised-clouds.md) | 35 min | – |

Engineers: 04, 05, 06 and 07 are the practical ones. Architects and consultants: everything.

## Completion checklist

- [ ] I can name the seven CAF phases and what each decides.
- [ ] I can draw the reference management group hierarchy from memory.
- [ ] I can explain why environments are not management groups.
- [ ] I can list the landing zone design areas.
- [ ] I can say where a Foundry workload sits and which policies should apply to it.
- [ ] I have written and assigned a custom policy and run a remediation task.
- [ ] I can write a compliant name for any common resource type.
- [ ] I can write a Resource Graph query that joins resources to their subscriptions.
- [ ] I can explain the difference between platform compliance and workload compliance.

## Maps to

- **AZ-900**: governance and compliance features.
- **AZ-104**: manage Azure governance (policy, locks, tags, management groups).
- **AZ-305**: design governance; design for compliance.
- **AB-100 / consulting**: adoption strategy, operating model, AI readiness.
- **SC-500**: policy-driven security posture.

## Next

[04 · Tooling and Infrastructure as Code](../04-tooling-and-infrastructure-as-code/README.md)
