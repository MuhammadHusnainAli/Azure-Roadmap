# Azure Roadmap

A complete, structured learning roadmap for Microsoft Azure, built for three roles:

- **Azure AI Engineer**: builds AI apps, RAG systems and agents on Microsoft Foundry.
- **AI Solution Architect**: designs secure, reliable, cost-aware AI and cloud solutions.
- **Azure Consultant**: turns business problems into scoped, justified, adoptable Azure solutions.

It covers the whole Azure platform as it stands in **October 2026**, with the deepest coverage on AI: Microsoft Foundry, Foundry Models, Azure AI Search and Foundry IQ, Foundry Agent Service, agent frameworks and protocols, evaluation, and AI governance.

> **Status:** the topic tree is defined in [Roadmap.md](Roadmap.md). Topic folders are being written one at a time.
>
> | Topic | State |
> |---|---|
> | [01 · Cloud and Azure Fundamentals](01-cloud-and-azure-fundamentals/README.md) | Written (9 of 9 subtopics) |
> | [02 · Identity and Access with Microsoft Entra](02-identity-and-access/README.md) | Written (11 of 11 subtopics) |
> | [03 · Governance and Landing Zones](03-governance-and-landing-zones/README.md) | Written (10 of 10 subtopics) |
> | [04 · Tooling and Infrastructure as Code](04-tooling-and-infrastructure-as-code/README.md) | Written (10 of 10 subtopics) |
> | [05 · Networking](05-networking/README.md) | Written (18 of 18 subtopics) |
> | [06 · Compute](06-compute/README.md) | Written (12 of 12 subtopics) |
> | 07 – 38 | Not started |

---

## Contents

- [What this roadmap covers](#what-this-roadmap-covers)
- [Repository structure](#repository-structure)
- [The roadmap at a glance](#the-roadmap-at-a-glance)
- [Role tracks](#role-tracks)
- [Suggested learning order](#suggested-learning-order)
- [Certification map](#certification-map)
- [What changed recently in Azure](#what-changed-recently-in-azure)
- [Subtopic file template](#subtopic-file-template)
- [Conventions](#conventions)
- [Keeping the roadmap current](#keeping-the-roadmap-current)
- [Sources](#sources)

---

## What this roadmap covers

| Area | Included |
|---|---|
| Foundations | Cloud concepts, resource hierarchy, Microsoft Entra identity, governance, landing zones, infrastructure as code |
| Core platform | Networking, compute, containers and Kubernetes, storage, databases, application platform, integration and messaging |
| Data | Microsoft Fabric, Azure Databricks, Data Factory, Synapse, Data Explorer, Microsoft Purview |
| AI | Generative AI foundations, Microsoft Foundry, Foundry Models and Azure OpenAI, prompt and context engineering, fine-tuning, RAG, agents, MCP and A2A, Foundry Tools, Azure Machine Learning, evaluation and GenAIOps, responsible AI and AI security, Copilot and Copilot Studio |
| Operations | Security, monitoring, DevOps, resiliency, FinOps, migration, hybrid and edge, IoT and industry services |
| Practice | Solution architecture, AI solution architecture, consulting practice, certifications, capstone projects |

**38 topics, 404 subtopics.** Every subtopic becomes one focused Markdown file.

It does not try to teach general programming, a specific language, or data science theory beyond what you need to build and design on Azure.

---

## Repository structure

```text
Azure-Roadmap/
├── README.md                              ← you are here
├── Roadmap.md                             ← full topic and subtopic tree
├── 01-cloud-and-azure-fundamentals/       ← one folder per topic
│   ├── README.md                          ← topic overview and reading order
│   ├── 01-cloud-computing-concepts.md     ← one file per subtopic
│   ├── 02-shared-responsibility-model.md
│   └── ...
├── 02-identity-and-access/
├── ...
└── 38-staying-current-and-legacy-services/
```

Rules for the structure:

- A `##` heading in `Roadmap.md` is a **topic** and maps to a numbered folder.
- A `###` heading in `Roadmap.md` is a **subtopic** and maps to a numbered `.md` file in that folder.
- Folder and file names are lowercase, hyphenated, and prefixed with a two-digit number so they sort in learning order.
- `Roadmap.md` is the source of truth. Change the tree there first, then the folders.

---

## The roadmap at a glance

**E** = AI Engineer, **A** = AI Solution Architect, **C** = Consultant. `●` core, `○` working knowledge, `–` awareness.

### Part A · Foundations

| # | Topic | E | A | C |
|---|---|:-:|:-:|:-:|
| 01 | Cloud and Azure Fundamentals | ● | ● | ● |
| 02 | Identity and Access with Microsoft Entra | ● | ● | ● |
| 03 | Governance and Landing Zones | ○ | ● | ● |
| 04 | Tooling and Infrastructure as Code | ● | ○ | ○ |

### Part B · Core Infrastructure and Application Platform

| # | Topic | E | A | C |
|---|---|:-:|:-:|:-:|
| 05 | Networking | ○ | ● | ○ |
| 06 | Compute | ○ | ● | ○ |
| 07 | Containers and Kubernetes | ● | ● | ○ |
| 08 | Storage | ● | ● | ○ |
| 09 | Databases | ● | ● | ○ |
| 10 | Application Platform | ● | ● | ○ |
| 11 | Integration, Messaging and APIs | ● | ● | ○ |

### Part C · Data and Analytics

| # | Topic | E | A | C |
|---|---|:-:|:-:|:-:|
| 12 | Data and Analytics | ○ | ● | ● |

### Part D · AI on Azure

| # | Topic | E | A | C |
|---|---|:-:|:-:|:-:|
| 13 | AI and Generative AI Foundations | ● | ● | ● |
| 14 | Microsoft Foundry Platform | ● | ● | ● |
| 15 | Foundry Models and Azure OpenAI | ● | ● | ○ |
| 16 | Prompt Engineering and Model Customisation | ● | ○ | ○ |
| 17 | Retrieval-Augmented Generation and Knowledge | ● | ● | ○ |
| 18 | Agents with Foundry Agent Service | ● | ● | ● |
| 19 | Agent Frameworks and Protocols | ● | ○ | – |
| 20 | Foundry Tools | ● | ○ | ○ |
| 21 | Azure Machine Learning and MLOps | ○ | ○ | – |
| 22 | Evaluation, Observability and GenAIOps | ● | ● | ○ |
| 23 | Responsible AI, AI Security and Governance | ● | ● | ● |
| 24 | Copilot, Low-Code Agents and Azure AI Assistants | ○ | ● | ● |

### Part E · Secure, Operate, Migrate and Extend

| # | Topic | E | A | C |
|---|---|:-:|:-:|:-:|
| 25 | Security | ○ | ● | ● |
| 26 | Monitoring and Observability | ● | ● | ○ |
| 27 | DevOps and Platform Engineering | ● | ○ | ○ |
| 28 | Resiliency, Backup and Disaster Recovery | ○ | ● | ○ |
| 29 | Cost Management and FinOps | ○ | ● | ● |
| 30 | Migration and Modernisation | – | ● | ● |
| 31 | Hybrid, Multicloud and Edge | – | ● | ○ |
| 32 | IoT, Industry and Specialised Services | – | ○ | ○ |

### Part F · Architecture, Consulting and Career

| # | Topic | E | A | C |
|---|---|:-:|:-:|:-:|
| 33 | Solution Architecture | ○ | ● | ● |
| 34 | AI Solution Architecture | ○ | ● | ● |
| 35 | Consulting Practice | – | ○ | ● |
| 36 | Certifications and Career | ● | ● | ● |
| 37 | Capstone Projects | ● | ● | ○ |
| 38 | Staying Current and Legacy Services | ○ | ● | ● |

The full subtopic list with scope notes is in [Roadmap.md](Roadmap.md).

---

## Role tracks

Everyone starts with topics **01, 02 and 13**. After that the paths diverge.

### Azure AI Engineer

You build and ship. Depth matters most in Part D.

| Stage | Topics | Outcome |
|---|---|---|
| 1. Foundations | 01, 02, 04 | Deploy and secure resources from code |
| 2. AI core | 13, 14, 15, 16 | Call models well, pick deployment types, control cost |
| 3. Knowledge | 08, 09, 17 | Build permission-aware RAG that you can measure |
| 4. Agents | 18, 19, 20 | Build single and multi-agent systems with tools, memory and MCP |
| 5. Hosting | 07, 10, 11 | Run AI apps on Container Apps, Functions and AKS behind an AI gateway |
| 6. Production | 22, 23, 26, 27 | Evaluate, trace, red team, and release through CI/CD |
| 7. Proof | 37 (projects 02 – 08), 36 | Portfolio and AI-103 |

### AI Solution Architect

You decide what gets built and why. Breadth across the platform, depth in design.

| Stage | Topics | Outcome |
|---|---|---|
| 1. Foundations | 01, 02, 03 | Design identity, governance and landing zones |
| 2. Platform breadth | 05 – 12 | Choose the right network, compute, data and integration services |
| 3. AI platform | 13, 14, 15, 17, 18 | Know what Foundry can and cannot do, and what it costs |
| 4. Trust | 23, 25, 28 | Design for security, compliance and failure |
| 5. Economics and operations | 22, 26, 29 | Capacity planning, FinOps and observability by design |
| 6. Architecture | 33, 34 | Produce reference-grade designs and defend the trade-offs |
| 7. Wider estate | 24, 30, 31 | Copilot build-vs-buy, migration, hybrid |
| 8. Proof | 37 (projects 01, 06, 09), 36 | AZ-305 and AB-100 |

### Azure Consultant

You connect business need to a deliverable solution. Breadth first, with enough depth to be credible.

| Stage | Topics | Outcome |
|---|---|---|
| 1. Foundations | 01, 02, 03, 13 | Speak the platform's language accurately |
| 2. AI landscape | 14, 18, 24 | Position Foundry, Copilot Studio and Microsoft 365 Copilot correctly |
| 3. Qualification | 12, 23, 29 | Assess data readiness, risk and cost |
| 4. Design literacy | 33, 34 | Read and challenge an architecture |
| 5. Engagement craft | 35, 30 | Run discovery, write proposals, estimate, drive adoption |
| 6. Proof | 37 (projects 09, 10), 36 | AZ-900, AI-901, then AB-100 |

---

## Suggested learning order

Assumes roughly 10 hours a week. Adjust to your background.

| Phase | Weeks | Focus | Topics |
|---|---|---|---|
| 1 | 1 – 4 | Azure foundations | 01 – 04 |
| 2 | 5 – 12 | Core platform | 05 – 11 |
| 3 | 13 – 15 | Data | 12 |
| 4 | 16 – 21 | AI core: models and RAG | 13 – 17 |
| 5 | 22 – 27 | Agents and AI services | 18 – 21 |
| 6 | 28 – 31 | Production AI: evaluation, security, governance | 22 – 24 |
| 7 | 32 – 38 | Operating Azure | 25 – 32 |
| 8 | 39 – 44 | Architecture and consulting | 33 – 35 |
| 9 | 45 – 52 | Capstones and certification | 36 – 38 |

Engineers can skim phases 3 and 7. Consultants can skim the hands-on parts of phases 2 and 5.

---

## Certification map

Microsoft replaced several long-standing exams in 2026. The table reflects what I found in October 2026, mostly from third-party certification sites, so **confirm each exam on Microsoft Learn before booking**.

| Exam | Credential | Replaces | Roadmap topics |
|---|---|---|---|
| AZ-900 | Azure Fundamentals | – | 01 – 03 |
| AI-901 | Azure AI Fundamentals | AI-900 (retired 30 Jun 2026) | 13, 14, 23 |
| AZ-104 | Azure Administrator Associate | – | 02 – 09, 26 |
| AI-103 | Azure AI Apps and Agents Developer Associate | AI-102 (retired 30 Jun 2026) | 14 – 23 |
| AI-200 | Azure AI Cloud Developer Associate | AZ-204 (retired 31 Jul 2026) | 07, 09 – 11 |
| AI-300 | MLOps Engineer Associate | DP-100 (retired 1 Jun 2026) | 21, 22 |
| AZ-305 | Azure Solutions Architect Expert | – | 03 – 12, 25 – 29, 33 |
| AB-100 | Agentic AI Business Solutions Architect | Pathway for several retired Power Platform and Dynamics exams | 18, 24, 34, 35 |
| SC-500 | Cloud and AI Security Engineer Associate | AZ-500 (retired 31 Aug 2026) | 23, 25 |
| AZ-400 | DevOps Engineer Expert | – | 27 |
| DP-700 | Fabric Data Engineer Associate | – | 12 |

Recommended sequence per role:

- **AI Engineer:** AZ-900 → AI-901 → AI-103 → AI-200 or AI-300
- **AI Solution Architect:** AZ-900 → AZ-104 → AI-103 → AZ-305 → AB-100
- **Consultant:** AZ-900 → AI-901 → AB-100, with AZ-305 if you lead technical design

---

## What changed recently in Azure

The roadmap is written against current names and services. If your prior knowledge is from 2024 or earlier, these are the shifts to know.

**Renames**

| Old name | Current name |
|---|---|
| Azure AI Studio, Azure AI Foundry | Microsoft Foundry |
| Azure AI services, Cognitive Services | Foundry Tools |
| Azure OpenAI Service | Azure OpenAI in Foundry Models |
| Azure AI Agent Service | Foundry Agent Service |
| Azure Active Directory | Microsoft Entra ID |
| Azure Stack HCI | Azure Local |
| Cosmos DB for MongoDB vCore | Azure DocumentDB |
| Azure Cache for Redis | Azure Managed Redis (successor) |

**Additions that shape the AI topics**

- **Foundry Agent Service** with prompt, workflow and hosted agents, memory, Toolboxes, routines, and publishing to Teams and Microsoft 365 Copilot.
- **Foundry IQ** as a unified knowledge layer over Azure AI Search, alongside Work IQ and Fabric IQ.
- **Foundry Control Plane**, **Microsoft Entra Agent ID** and **Agent 365** for governing fleets of agents.
- **Microsoft Agent Framework**, the successor to Semantic Kernel and AutoGen.
- **Model Context Protocol** and **Agent-to-Agent** as the standard tool and agent interop protocols.
- A much wider **model catalogue**: OpenAI, Anthropic Claude, Microsoft MAI, Phi, Meta, Mistral, xAI and more, with a model router.
- **Azure Copilot** and **Azure SRE Agent** for agentic operations.

Many of these have features still in preview, and secondary sources disagree on some GA dates. Each subtopic file will state the status as checked against Microsoft Learn at the time of writing.

Topic 38 holds the full rename map and the retired-services list.

---

## Subtopic file template

Every subtopic file follows the same shape so the roadmap reads consistently.

```markdown
# <Subtopic title>

> Topic: <NN · Topic name> · Tracks: E ● A ○ C – · Status checked: <YYYY-MM>

## Why it matters
Two or three sentences on where this shows up in real work.

## Key concepts
The ideas to understand, explained plainly.

## How it works on Azure
Services, tiers, limits and configuration that matter.

## Design decisions and trade-offs
The choices an architect has to make, with guidance.

## Hands-on
A short lab: portal steps, CLI, Bicep or SDK code.

## Security, cost and reliability notes
What to watch for in production.

## Common mistakes
What goes wrong and how to avoid it.

## Check yourself
Five to ten questions, exam and interview style.

## References
Links to Microsoft Learn and the Azure Architecture Center.
```

Each topic folder also gets a `README.md` with an overview, prerequisites, reading order and a completion checklist.

---

## Conventions

- **Current names first.** Former names appear once in brackets.
- **Status labels.** Features are marked `GA`, `Preview` or `Retiring`, with the month the status was checked.
- **Decision guides.** Topics with overlapping services include a "which one do I pick" file.
- **Code.** Examples use Azure CLI and Bicep for infrastructure, and Python for SDK work, unless a topic needs otherwise.
- **Keyless by default.** Examples authenticate with Microsoft Entra ID and managed identities, not keys.
- **Links.** Microsoft Learn is the primary reference.

---

## Keeping the roadmap current

Azure changes weekly. To keep this accurate:

1. Review [Azure Updates](https://azure.microsoft.com/updates/) and the Foundry what's-new page monthly.
2. Revisit the tree after **Microsoft Build** (May) and **Microsoft Ignite** (November). Ignite 2026 is close and will likely change parts of topics 14 – 24.
3. When a service is renamed or retired, update `Roadmap.md` first, then topic 38, then the affected files.
4. Bump the "Status checked" date on any file you verify.

---

## Sources

Used to establish the October 2026 service landscape and certification changes:

- [Azure products directory](https://azure.microsoft.com/en-us/products/)
- [Azure documentation hub](https://learn.microsoft.com/en-us/azure/?product=popular)
- [What's new in Microsoft Foundry: Build 2026](https://devblogs.microsoft.com/foundry/whats-new-in-microsoft-foundry-build-2026/)
- [Foundry IQ announcement](https://devblogs.microsoft.com/foundry/build-smarter-agents-faster-with-foundry-iq/)
- [Foundry Control Plane and Agent 365](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/foundry-control-plane-and-agent-365-two-control-planes-walk-into-an-enterprise/4520185)
- [Microsoft Agent 365 integration with Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-365-integration)
- [Microsoft Credentials roundup: June 2026](https://techcommunity.microsoft.com/blog/skills-hub-blog/microsoft-credentials-roundup-june-2026/4528350)
- [Microsoft certification retirements 2026](https://readroo.st/blog/microsoft-certification-retirements-2026) (third party)
- [Azure weekly update, 9 October 2026](https://daily.dev/posts/azure-weekly-update---9th-october-2026-hib0vmnoy) (third party)
