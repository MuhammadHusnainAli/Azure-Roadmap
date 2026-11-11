# AI Regulation and Standards

## Why it matters

AI is now regulated. The **EU AI Act** applies to anyone placing AI systems on the EU market or using them in the EU, with fines up to 7% of global turnover for prohibited practices. Sector regulators (financial services, health, employment) add their own expectations, and customers increasingly ask for **ISO/IEC 42001** certification or alignment with the **NIST AI Risk Management Framework** in procurement. Architects and consultants don't need to be lawyers, but they must be able to classify a use case, explain which obligations follow, and map those obligations to concrete technical and organisational controls on Azure. This file gives you that map. It isn't legal advice: confirm specifics with counsel and the official texts.

## Key concepts

### EU AI Act (Regulation (EU) 2024/1689)

**Risk-based tiers**

| Tier | Examples | Consequence |
|---|---|---|
| **Prohibited** | Social scoring, manipulative techniques causing harm, untargeted facial image scraping, emotion recognition in workplaces and schools (with exceptions), certain biometric categorisation and real-time remote biometric identification for law enforcement (narrow exceptions) | Banned |
| **High-risk** | Annex III: employment and worker management, access to education, credit scoring, essential public and private services, law enforcement, migration, justice, critical infrastructure; Annex I: safety components of regulated products | Conformity obligations (below) |
| **Transparency (limited risk)** | Chatbots, emotion recognition, deepfakes and AI-generated content | Disclose AI interaction; mark synthetic content |
| **Minimal risk** | Spam filters, most internal productivity tools | Voluntary codes; AI literacy duty still applies |
| **General-purpose AI (GPAI) models** | Foundation models; systemic-risk models above compute thresholds | Provider duties: documentation, copyright policy, training data summary; extra for systemic risk |

**High-risk obligations (providers, with deployer duties)**: risk management system, data governance, technical documentation, record-keeping (logging), transparency and instructions for use, human oversight, accuracy/robustness/cybersecurity, quality management, conformity assessment, registration, post-market monitoring and serious-incident reporting. **Deployers** must use systems per instructions, assign competent human oversight, monitor operation, keep logs, inform affected workers, and in some cases perform a fundamental rights impact assessment.

**Roles**: provider (develops or places on the market under its name), deployer (uses under its authority), importer, distributor. Building an agent on Azure OpenAI usually makes **you** the provider of your AI system and Microsoft a provider of the GPAI model and platform. Substantially modifying a system can make you its provider.

**Timeline** (as amended by the AI Omnibus, Regulation (EU) 2026/1744, in force 27 July 2026; verify current dates):

| Date | What applies |
|---|---|
| 2 February 2025 | Prohibitions and AI literacy |
| 2 August 2025 | GPAI model obligations; governance and penalties |
| 2 August 2026 | Most remaining provisions, including transparency obligations (some content-marking duties have until 2 December 2026) |
| 2 December 2027 | High-risk systems in Annex III (postponed from August 2026) |
| 2 August 2028 | High-risk systems in Annex I products (postponed from August 2027) |

### NIST AI Risk Management Framework (AI RMF 1.0)

Voluntary, widely adopted (especially in the US), with a **Generative AI Profile** (NIST AI 600-1).

| Function | Purpose | Azure examples |
|---|---|---|
| **Govern** | Culture, policies, accountability, roles | AI governance board, policies, Azure Policy, Purview Compliance Manager |
| **Map** | Context, intended use, risks and impacts | Use-case qualification, impact assessment, threat model |
| **Measure** | Analyse, assess, track risks | Foundry evaluations, red teaming, RAI dashboard |
| **Manage** | Prioritise, respond, monitor | Guardrails, monitoring, incident response, decommissioning |

### ISO/IEC 42001:2023

The first certifiable **AI management system (AIMS)** standard, built like ISO 27001: context, leadership, planning (AI risk assessment, AI impact assessment), support, operation, performance evaluation, improvement, with Annex A controls (policies, internal organisation, resources, impact assessment, lifecycle, data, information for interested parties, use of AI systems, third-party relationships). Microsoft has ISO/IEC 42001 certification for in-scope AI services (including Azure AI services and Microsoft 365 Copilot); your organisation still needs its own AIMS if you want certification.

Related standards: ISO/IEC 23894 (AI risk management), ISO/IEC 22989 (concepts and terminology), ISO/IEC 5338 (AI lifecycle).

### Other regimes to know

| Regime | Note |
|---|---|
| **GDPR** | Lawful basis, DPIAs, data subject rights, Article 22 on solely automated decisions |
| **UK** | Principles-based, sector-regulator approach (ICO, FCA, CMA guidance) |
| **US** | State laws (for example Colorado AI Act), sector rules, federal guidance |
| **Sector** | Financial model risk management (for example SR 11-7, PRA SS1/23), medical device regulation |

## How it works on Azure

Map obligations to controls once and reuse them:

| Obligation | Control | Evidence |
|---|---|---|
| Risk management | Impact assessment, threat model, risk register (files 01–02) | Documents with sign-off |
| Data governance | Purview classification, lineage, data quality checks (file 05) | Data map, quality reports |
| Technical documentation | Agent specification, model cards, architecture records | Versioned docs in Git |
| Logging / record-keeping | Tracing, audit logs, Purview audit (Topic 22 file 06) | Retained logs per policy |
| Transparency | AI disclosure in UX, content credentials on generated images | Screenshots, configuration |
| Human oversight | Approval gates, escalation, override (file 04) | Process and logs |
| Accuracy, robustness | Evaluations, CI gates (Topic 22) | Evaluation reports |
| Cybersecurity | Guardrails, Defender for Cloud, red teaming (files 03, 06) | Scan results, alerts |
| Post-market monitoring | Continuous evaluation, incident process | Dashboards, incident records |
| AI literacy | Training programme | Completion records |

**Purview Compliance Manager** provides assessment templates for the EU AI Act, ISO/IEC 42001 and NIST AI RMF, mapping requirements to Microsoft-managed and customer-managed controls with improvement actions ([Topic 03 file 08](../03-governance-and-landing-zones/08-regulatory-compliance.md)). The **Service Trust Portal** provides Microsoft's audit reports, including ISO/IEC 42001.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Classify early** | Classify each use case at intake (file 09); a high-risk classification changes timeline and cost significantly. |
| **One control framework** | Map EU AI Act, NIST and ISO 42001 onto one internal control set to avoid duplicate work. |
| **Provider vs deployer** | Clarify your role per system; contracts with vendors should allocate obligations. |
| **Design to avoid high-risk** | Sometimes scope changes (decision support with human decision vs automated decision) change classification; get legal confirmation. |
| **Certification** | ISO 42001 certification when customers demand it; alignment is often enough internally. |
| **Global programmes** | Use the strictest common denominator for shared platforms; localise where laws differ. |

## Hands-on

1. Classify five example use cases (HR CV screening, internal IT helpdesk agent, credit pre-approval, marketing image generator, customer chatbot) under the EU AI Act.
2. For the high-risk one, list provider and deployer obligations and map each to an Azure control and evidence.
3. Open Compliance Manager, add the EU AI Act and ISO/IEC 42001 assessments, review improvement actions.
4. Download Microsoft's ISO/IEC 42001 certificate from the Service Trust Portal.
5. Write an AI literacy plan for the teams building and using your agents.

## Security, cost and reliability notes

- **Compliance is continuous**: models, prompts and data change; re-assess on material changes.
- **Logging obligations conflict with minimisation**: design retention that satisfies both (pseudonymisation, tiered retention).
- **Cost**: high-risk conformity work (documentation, testing, oversight) is substantial; include it in business cases.

## Common mistakes

| Mistake | Fix |
|---|---|
| Assuming "we use Microsoft, so we're compliant" | Microsoft covers its role; you own your system's obligations |
| Classifying only at launch | Classify at intake and on change |
| Separate projects for each framework | One control set mapped to many frameworks |
| Forgetting deployer obligations | Oversight, monitoring, logs, worker information |
| Out-of-date dates | Check official sources; the AI Omnibus moved high-risk deadlines |

## Check yourself

1. Describe the EU AI Act's risk tiers with one example each.
2. List five obligations for high-risk AI systems.
3. Which dates apply to prohibitions, GPAI and Annex III high-risk systems?
4. What are the four NIST AI RMF functions?
5. What is ISO/IEC 42001, and does Microsoft's certification make you compliant?
6. Which Azure tool provides regulatory assessment templates?

<details>
<summary>Answers</summary>

1. Prohibited (social scoring), high-risk (CV screening, credit scoring), transparency (chatbots, deepfakes), minimal (spam filters); plus GPAI model obligations.
2. Any five: risk management, data governance, technical documentation, logging, transparency/instructions, human oversight, accuracy/robustness/cybersecurity, quality management, conformity assessment, registration, post-market monitoring.
3. Prohibitions from 2 February 2025; GPAI from 2 August 2025; Annex III high-risk from 2 December 2027 after the AI Omnibus (verify current dates).
4. Govern, Map, Measure, Manage.
5. A certifiable AI management system standard; Microsoft's certification covers its services, not your organisation's AIMS or systems.
6. Microsoft Purview Compliance Manager.

</details>

## References

- [EU AI Act, Regulation (EU) 2024/1689 (EUR-Lex)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj)
- [European Commission AI regulatory framework](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [ISO/IEC 42001 offering (Microsoft)](https://learn.microsoft.com/compliance/regulatory/offering-iso-42001)
- [Compliance Manager regulations list](https://learn.microsoft.com/purview/compliance-manager-regulations-list)
- [Govern AI (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/govern)
