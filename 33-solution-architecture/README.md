# 33 · Solution Architecture

> Tracks: E ○ A ● C ● · Part F · Architecture, Consulting and Career

This topic turns service knowledge into architecture practice. It covers the Azure Well-Architected Framework and its five pillars; running Well-Architected Reviews with evidence and service guides; capturing requirements, quality attributes, constraints and assumptions; cloud design patterns for resilience, data, gateways and integration; architecture styles from N-tier to microservices and event-driven; multitenant SaaS architecture; mission-critical design with stamps and health modelling; architecture decision records and diagrams (C4 and Azure icons); using the Azure Architecture Center; and consolidated technology decision guides.

## What you will be able to do

- Apply WAF pillars and make trade-offs explicit.
- Run a Well-Architected Review and produce a prioritised roadmap.
- Turn vague goals into measurable requirements and constraints.
- Apply cloud design patterns and choose architecture styles.
- Design multitenant and mission-critical systems.
- Record decisions and communicate designs with ADRs and diagrams.
- Find and adapt reference architectures.
- Choose Azure services with defensible decision criteria.

## Prerequisites

Parts A–E of the roadmap (or equivalent experience), especially [Topic 03](../03-governance-and-landing-zones/README.md), [Topic 25](../25-security/README.md), [Topic 28](../28-resiliency-backup-and-disaster-recovery/README.md) and [Topic 29](../29-cost-management-and-finops/README.md).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Azure Well-Architected Framework](01-azure-well-architected-framework.md) | 35 min | Pillar targets and trade-offs |
| 02 | [Well-Architected Reviews and Service Guides](02-well-architected-reviews-and-service-guides.md) | 40 min | Run an assessment |
| 03 | [Requirements and Constraints](03-requirements-and-constraints.md) | 35 min | NFR workshop |
| 04 | [Cloud Design Patterns](04-cloud-design-patterns.md) | 45 min | Retry, circuit breaker, cache-aside |
| 05 | [Architecture Styles](05-architecture-styles.md) | 35 min | Compare styles for one system |
| 06 | [Multitenant Architecture](06-multitenant-architecture.md) | 40 min | Tenancy model, RLS |
| 07 | [Mission-Critical Design](07-mission-critical-design.md) | 40 min | Health endpoint, blue-green stamps |
| 08 | [Architecture Decision Records and Diagrams](08-architecture-decision-records-and-diagrams.md) | 30 min | ADRs, C4 diagrams |
| 09 | [Azure Architecture Center](09-azure-architecture-center.md) | 30 min | Deploy a baseline |
| 10 | [Technology Decision Guides](10-technology-decision-guides.md) | 35 min | Five scenario decisions |

**Architects and consultants:** all of it. **AI engineers:** 01, 04, 05, 08 and 10.

## Completion checklist

- [ ] I can explain the five pillars and their trade-offs.
- [ ] I have run a Well-Architected Review with evidence and a roadmap.
- [ ] I write measurable NFRs and track assumptions.
- [ ] I apply resilience and data patterns correctly.
- [ ] I can justify an architecture style choice.
- [ ] I can design tenant isolation and per-tenant metering.
- [ ] I can explain stamps, health modelling and zero-downtime deployment.
- [ ] I keep ADRs and C4/deployment diagrams in source control.
- [ ] I adapt Architecture Center baselines with recorded deviations.
- [ ] I choose services using decision guides and lifecycle checks.

## Maps to

- **AZ-305**: Azure Solutions Architect Expert (design across identity, data, business continuity, infrastructure).
- **AB-100** (adjacent): solution architecture for agentic business solutions.

## Next

34 · [AI Solution Architecture](../34-ai-solution-architecture/README.md)
