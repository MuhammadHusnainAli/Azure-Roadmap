# Portfolio, Interviews and Career Paths

## Why it matters

Certifications get you past CV filters. Portfolios and interviews get you hired or win you engagements. Employers and clients want evidence that you've built, secured, evaluated and operated real AI solutions on Azure, and that you can explain the trade-offs you made. This file covers what to build, how to present it, and a question bank for each of the three roles in this roadmap, plus how those roles typically progress.

## Key concepts

### What to build

| Track | Portfolio pieces (from [Topic 37](../37-capstone-projects/README.md)) | Evidence to include |
|---|---|---|
| **Azure AI Engineer** | Enterprise RAG assistant (02), multi-agent process (03), GenAIOps pipeline (08), MCP server (07) | Repo, IaC, evaluation results, tracing screenshots, cost per request |
| **AI Solution Architect** | Secure landing zone (01), generative AI gateway (06), architecture review (09) | Architecture diagrams, ADRs, WAF assessment, cost model, threat model |
| **Azure Consultant** | Client engagement simulation (10), architecture review (09), RAG assistant (02) as a demo | Discovery notes, business case, proposal, SoW, executive readout deck |

### Presenting a project

Use a consistent **README structure** for each portfolio repo:

1. Problem and users.
2. Architecture diagram.
3. Key decisions (link to ADRs).
4. Security approach (identity, network, data protection, guardrails).
5. Quality evidence (evaluation metrics, tests).
6. Operations (monitoring, alerts, runbooks).
7. Cost (estimate and actuals).
8. What I'd do next.
9. How to deploy (`azd up` or pipeline).

Never publish client data, secrets or proprietary code. Use synthetic or public datasets.

### Telling the story: STAR plus trade-offs

**Situation, Task, Action, Result**, then add **Trade-offs** (what you rejected and why) and **Learning** (what you'd change). Interviewers for architecture and AI roles probe trade-offs more than outcomes.

### Career paths

| Track | Typical progression |
|---|---|
| **Engineer** | Cloud or software engineer → AI engineer → senior or lead AI engineer → AI platform engineer or principal engineer |
| **Architect** | Engineer or administrator → cloud solution architect → AI solution architect → enterprise or chief architect |
| **Consultant** | Technical consultant → senior consultant → principal or managing consultant → practice lead or partner |

Transitions between tracks are common. Engineers move into architecture by owning designs and ADRs. Architects move into consulting by leading discovery and business cases.

## How it works on Azure

### Interview question bank

**Azure AI Engineer**

1. Walk me through how a request flows through your RAG app, from user to answer.
2. How do you evaluate groundedness and relevance? What thresholds do you gate on?
3. How do you secure an agent that can call tools with write access?
4. How would you reduce token cost by 40% without hurting quality?
5. Explain how you trace a multi-agent workflow and find the failing step.
6. When would you fine-tune instead of using RAG or prompting?
7. How do you handle 429 errors and quota limits in production?
8. How do you authenticate to Foundry without keys?

**AI Solution Architect**

1. Design an enterprise AI platform for 20 business units. How do you handle quotas, chargeback and governance?
2. How do you make a chat application resilient to a regional model outage?
3. Where do you enforce per-user authorisation in a RAG system?
4. Compare provisioned and standard deployments for a high-volume workload.
5. How would you design a network-isolated Foundry environment?
6. When is a multi-agent design justified over a single agent?
7. How do you document and communicate an architecture decision?
8. What are the main WAF trade-offs for AI workloads?

**Azure Consultant**

1. How do you run a discovery workshop for a client new to AI?
2. How do you prioritise twenty use cases?
3. Build a business case for an AI assistant in front of a CFO.
4. A PoC succeeded but the client is stuck. What do you do?
5. How do you estimate and scope a project when model quality is uncertain?
6. How do you explain hallucination and prompt injection to an executive?
7. How do you drive adoption of Microsoft 365 Copilot after licences are bought?
8. How would you handle a client who wants to skip security review to hit a date?

### Practical interview tasks

- **Whiteboard design** (45–60 minutes): clarify requirements, then cover the architecture, security, reliability, cost and operations.
- **Code exercise**: build a small RAG or agent endpoint, or debug one.
- **Case study** (consulting): read a brief, then present recommendations and a plan.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Depth vs breadth in portfolio** | Two deep, production-quality projects beat ten demos. |
| **Public vs private** | Public repos with synthetic data; describe client work only at a level your contract allows. |
| **Specialise vs generalise** | Generalise early (platform foundations), then specialise (agents, AI platform, industry). |
| **Community presence** | Blog posts, talks and community contributions build reputation; keep content accurate and dated. |

## Hands-on

1. Pick two capstones from topic 37 that fit your track and build them.
2. Write a portfolio README for each, using the structure above.
3. Prepare five STAR-plus-trade-offs stories.
4. Answer the question bank for your track aloud and record yourself.
5. Do a mock whiteboard design with a peer and ask for feedback.

## Security, cost and reliability notes

- Scan portfolio repos for secrets before publishing; enable GitHub secret scanning and push protection.
- Use `azd down` or delete resource groups; don't leave demos running on your card.
- Make demo apps fail gracefully with clear messages when resources are stopped.

## Common mistakes

| Mistake | Fix |
|---|---|
| Portfolio of tutorial clones | Add your own requirements, evaluation and security |
| No evidence of quality | Show evaluation metrics and tests |
| Talking only about technology | Explain business outcome and trade-offs |
| Publishing secrets or client data | Synthetic data and secret scanning |
| Ignoring soft skills | Practise communication and stakeholder stories |

## Check yourself

1. Which capstones suit each track's portfolio?
2. What sections should a portfolio README include?
3. What does STAR plus trade-offs add to a standard STAR answer?
4. Name three typical interview tasks.
5. How do engineers move into architecture roles?

<details>
<summary>Answers</summary>

1. Engineer: 02, 03, 07, 08; architect: 01, 06, 09; consultant: 10, 09 and 02 as a demo.
2. Problem and users, architecture, key decisions, security, quality evidence, operations, cost, next steps, deployment.
3. The rejected alternatives and the reasons, plus what you learned.
4. Whiteboard design, code exercise and case study presentation.
5. By owning designs, writing ADRs, leading reviews and broadening across platform areas.

</details>

## References

- [Microsoft Learn career paths](https://learn.microsoft.com/training/career-paths/)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/overview)
- [GitHub secret scanning](https://docs.github.com/code-security/secret-scanning/introduction/about-secret-scanning)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
