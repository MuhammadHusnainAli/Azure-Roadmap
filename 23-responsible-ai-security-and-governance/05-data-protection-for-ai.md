# Data Protection for AI

## Why it matters

Generative AI doesn't create new data access, but it makes existing access frightfully efficient. A user who technically had access to a badly permissioned HR folder never found it; Microsoft 365 Copilot or a RAG agent will summarise it for them in seconds. Users also paste confidential data into prompts, including into unsanctioned consumer AI sites. **Microsoft Purview** extends the information protection, DLP, audit, retention and insider risk controls organisations already use to AI apps and agents, and **Data Security Posture Management (DSPM)** gives a single place to discover AI usage, oversharing and sensitive data in prompts. This is the data half of the "Protect" stage and a frequent blocker or enabler for Copilot and agent rollouts.

## Key concepts

### The AI data risks

| Risk | Example |
|---|---|
| **Oversharing** | SharePoint sites shared with "Everyone except external users" surfaced by Copilot or agents |
| **Sensitive data in prompts** | Users paste customer lists into a third-party AI site |
| **Sensitive data in responses** | Agent returns another customer's data or labelled content to someone without rights |
| **Shadow AI** | Unsanctioned AI apps and agents processing corporate data |
| **Retention and discovery** | Prompts and responses are records subject to retention, legal hold and eDiscovery |
| **Insider risk** | Users probing AI for protected material or attempting prompt injection |

### DSPM and DSPM for AI

Purview's new unified **DSPM** (Preview, incorporating AI apps and agents) and **DSPM for AI (classic)** provide:

| Capability | What it does |
|---|---|
| **Insights and analytics** | AI activity across Microsoft 365 Copilot, agents, enterprise AI apps (including Foundry apps) and third-party AI sites |
| **One-click policies** | Preconfigured collection, DLP and insider risk policies (for example, detect sensitive info shared with AI, capture interactions for enterprise AI apps) |
| **Data risk assessments** | Weekly automatic assessment of the top 100 SharePoint sites by usage for oversharing; custom assessments (Preview) |
| **Recommendations** | Create sensitivity labels, protect labelled items from Copilot processing, detect risky AI usage, secure Azure AI apps and agents, regulatory guidance via Compliance Manager |
| **Apps and agents view** | Inventory of AI apps and agents with the sensitive data they accessed |

Prerequisites include Purview Audit (on by default for new tenants), and for third-party AI sites, device onboarding and the Purview browser extension.

### Purview capabilities for Microsoft Foundry

| Capability | Supported for Foundry AI interactions |
|---|---|
| DSPM / DSPM for AI | ✓ |
| Audit (prompts and responses in the unified audit log) | ✓ |
| Data classification (sensitive info types, trainable classifiers) | ✓ |
| Sensitivity labels | ✓ |
| DLP | ✓ (blocking prompts by sensitive info type for Entra-registered apps integrated with Purview APIs) |
| Insider Risk Management (Risky AI usage template) | ✓ |
| Communication compliance | ✓ |
| eDiscovery | ✓ |
| Data Lifecycle Management (retention of prompts/responses) | ✓ |
| Compliance Manager | ✓ |
| Encryption without sensitivity labels | ✕ |

Enable it either in the **Foundry Control Plane** or via **Defender for Cloud** ("Enable data security for Azure AI with Microsoft Purview"). Policy management for Foundry interactions needs Purview **pay-as-you-go** billing; audit is included in the Purview licence. Data security policies apply when API calls carry **user context** (Entra user token or explicitly supplied user context); otherwise interactions appear in audit and activity explorer only.

### Sensitivity labels and AI

- AI apps supported by Purview honour existing permissions: data the user can't access isn't returned or used.
- For **encrypted** labelled content, the user needs both **VIEW** and **EXTRACT** usage rights for AI to return it.
- A DLP policy can **exclude labelled items from Microsoft 365 Copilot and agent processing** (for example, "Highly Confidential").
- Copilot responses inherit the most restrictive label of referenced content.

### Oversharing controls (Microsoft 365)

| Control | Purpose |
|---|---|
| DSPM data risk assessments | Find overshared sites and items |
| SharePoint Advanced Management: **Restricted Content Discovery**, **Restricted Access Control**, site access reviews | Hide sites from Copilot/org-wide search; limit site access to groups |
| Sensitivity labels with default site/library labels | Classify and protect at scale |
| Retention and deletion of stale content | Reduce what AI can surface |

## How it works on Azure

For a Foundry-based agent or RAG app:

1. **Classify sources**: scan Azure Storage, SQL and other sources with Purview Data Map; apply labels.
2. **Respect permissions at retrieval**: Azure AI Search security trimming or Foundry IQ permission-aware retrieval, and on-behalf-of for SharePoint ([Topic 17](../17-rag-and-knowledge/README.md)).
3. **Pass user context** in API calls so Purview policies and Defender alerts can attribute interactions to users.
4. **Enable Purview for Foundry** (Control Plane or Defender for Cloud); turn on DSPM recommendations "Secure data in Azure AI apps and agents".
5. **Guardrail PII** controls for model or tool-call outputs (file 03).
6. **Retention**: Purview retention policies for AI interactions, plus Application Insights retention for traces (file 07).
7. **Monitor**: DSPM reports, activity explorer, insider risk alerts flowing into Defender XDR.

For Microsoft 365 Copilot and Copilot Studio agents, the same Purview controls apply natively ([Topic 24](../24-copilot-and-low-code-agents/README.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Fix oversharing before Copilot** | Run assessments, apply Restricted Content Discovery on risky sites as a short-term control while permissions are remediated. |
| **Label taxonomy** | Keep it simple (4–5 labels); auto-label high-risk content. |
| **Exclude from AI** | Use the "exclude labelled content from Copilot processing" DLP only for genuinely restricted content; overuse reduces value. |
| **Third-party AI** | Endpoint DLP and network DLP to block sensitive pastes; offer a sanctioned alternative. |
| **User context** | Design apps to call models with user tokens or supply user context so policies can apply. |
| **Retention** | Align prompt/response retention with records policy and privacy minimisation. |

## Hands-on

1. In the Purview portal, open DSPM, review AI apps and agents, and activate two one-click policies.
2. Review the default oversharing assessment; remediate one overshared site and apply Restricted Content Discovery to another.
3. Create a DLP policy preventing Copilot from processing "Highly Confidential" content; test.
4. Enable Purview data security for a Foundry project; send prompts with user context containing a test credit card number; find the interaction in activity explorer.
5. Configure a retention policy for AI interactions and an insider risk policy using the Risky AI usage template.

## Security, cost and reliability notes

- **Prompts are data**: they're logged, retained and discoverable; tell users.
- **Licensing and billing**: some capabilities need specific Microsoft 365 E5/Purview licences; Foundry policy management uses pay-as-you-go.
- **Latency**: assessments and reports take at least a day to populate.
- **Permissions are the foundation**: Purview won't fix a bad permission model; it helps you find and contain it.

## Common mistakes

| Mistake | Fix |
|---|---|
| Rolling out Copilot without an oversharing assessment | Assess and remediate first |
| RAG index built with a service identity and no trimming | Security trimming or OBO |
| Calling models without user context | Pass user context so policies apply |
| Blocking all AI sites with no alternative | Provide sanctioned AI plus DLP |
| Forgetting retention and eDiscovery for prompts | Retention policies for AI interactions |

## Check yourself

1. Why does generative AI amplify oversharing risk?
2. What are DSPM for AI's four capability groups?
3. Which Purview capabilities support Foundry interactions, and which doesn't?
4. What two ways can you enable Purview for Foundry?
5. When do Purview data security policies apply to Foundry API calls?
6. What usage rights does AI need for encrypted labelled content?

<details>
<summary>Answers</summary>

1. AI efficiently finds and summarises anything the user can technically access, surfacing content that was previously obscure.
2. Insights and analytics, one-click policies, data risk assessments, compliance controls (plus recommendations and the apps/agents view).
3. DSPM, audit, classification, labels, DLP, insider risk, communication compliance, eDiscovery, lifecycle management, Compliance Manager; encryption without labels isn't supported.
4. In the Foundry Control Plane, or in Defender for Cloud.
5. When calls use an Entra user-context token or explicitly include user context; otherwise only audit and activity explorer.
6. VIEW and EXTRACT.

</details>

## References

- [Purview for Microsoft Foundry](https://learn.microsoft.com/purview/ai-azure-foundry)
- [Microsoft Purview for AI](https://learn.microsoft.com/purview/ai-microsoft-purview)
- [DSPM for AI (classic)](https://learn.microsoft.com/purview/dspm-for-ai)
- [Govern and secure AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization)
