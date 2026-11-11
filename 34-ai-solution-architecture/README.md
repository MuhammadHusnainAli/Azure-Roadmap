# 34 · AI Solution Architecture

> Tracks: E ○ A ● C ● · Part F · Architecture, Consulting and Career

This topic assembles the AI services from Parts D and E into production architectures. It covers Well-Architected guidance for AI workloads; the baseline Microsoft Foundry chat reference architecture component by component; enterprise RAG with ingestion, retrieval, agentic retrieval and security trimming; agentic architectures and orchestration patterns (and when not to use them); generative AI gateways; network-isolated AI in landing zones; multi-region and high-availability AI with capacity and data-zone constraints; multitenant AI isolation; capacity and performance planning; document processing and voice/contact centre designs; integrating agents with SAP, Dynamics 365, ServiceNow and Salesforce; and the AI platform operating model.

## What you will be able to do

- Apply WAF AI principles and design areas to AI workloads.
- Explain and adapt the baseline Foundry chat architecture.
- Design enterprise RAG with permissions, freshness and evaluation.
- Choose the right level of agent complexity and orchestration pattern.
- Design gateways, private networking and multi-region resilience for AI.
- Isolate tenants across models, indexes, memory and agents.
- Plan capacity and latency with token-based models and load tests.
- Design IDP and voice solutions and integrate agents with systems of record.
- Define an AI platform and CoE operating model.

## Prerequisites

Topics [14](../14-microsoft-foundry-platform/README.md)–[23](../23-responsible-ai-security-and-governance/README.md) and [33](../33-solution-architecture/README.md).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Well-Architected for AI Workloads](01-well-architected-for-ai-workloads.md) | 35 min | WAF AI assessment |
| 02 | [Baseline Foundry Chat Reference Architecture](02-baseline-foundry-chat-reference-architecture.md) | 50 min | Deploy reference implementation |
| 03 | [Enterprise RAG Architecture](03-enterprise-rag-architecture.md) | 45 min | Security-trimmed RAG |
| 04 | [Agentic Architecture](04-agentic-architecture.md) | 45 min | Sequential vs concurrent |
| 05 | [Generative AI Gateway Architecture](05-generative-ai-gateway-architecture.md) | 40 min | Backend pools, token limits |
| 06 | [Network-Isolated AI Architecture](06-network-isolated-ai-architecture.md) | 45 min | Private Foundry and agents |
| 07 | [Multi-Region and High-Availability AI](07-multi-region-and-high-availability-ai.md) | 40 min | Cross-region failover |
| 08 | [Multitenant AI Solutions](08-multitenant-ai-solutions.md) | 40 min | Tenant-scoped tools |
| 09 | [Capacity and Performance Planning](09-capacity-and-performance-planning.md) | 40 min | Token model, load test |
| 10 | [Document and Voice Solution Architectures](10-document-and-voice-solution-architectures.md) | 45 min | IDP flow, voice agent |
| 11 | [Integrating AI with Enterprise Systems](11-integrating-ai-with-enterprise-systems.md) | 40 min | REST API as MCP server |
| 12 | [AI Platform Operating Model](12-ai-platform-operating-model.md) | 35 min | Operating model draft |

**Architects:** all of it. **Consultants:** 01, 02, 04, 05, 10, 11 and 12. **AI engineers:** 02–09 and 11.

## Completion checklist

- [ ] I can map an AI workload to WAF AI design areas.
- [ ] I can explain every component of the baseline Foundry chat architecture.
- [ ] I have built permission-aware RAG with evaluation.
- [ ] I can justify single vs multi-agent designs with evidence.
- [ ] I have configured a gateway with failover and token limits.
- [ ] I can design a fully private AI workload in a landing zone.
- [ ] I can design multi-region AI within residency constraints.
- [ ] I can isolate tenants in AI solutions and test for leakage.
- [ ] I have produced a capacity plan and load-tested an AI endpoint.
- [ ] I can design IDP and voice solutions with human-in-the-loop.
- [ ] I can expose enterprise systems as safe agent tools.
- [ ] I can propose an AI platform operating model.

## Maps to

- **AI-103**: design and implement AI solutions on Foundry.
- **AB-100**: agentic AI business solution architecture.
- **AZ-305** (adjacent): solution design.

## Next

35 · [Consulting Practice](../35-consulting-practice/README.md)
