# 23 · Responsible AI, AI Security and Governance

> Tracks: E ● A ● C ● · Part D · AI on Azure

AI systems create new kinds of harm and new attack surfaces, and they are now regulated. This topic covers how to build and run AI responsibly and securely on Azure: Microsoft's responsible AI principles, impact assessments and transparency notes; the AI threat landscape including direct and indirect prompt injection and the OWASP Top 10 lists for LLMs and agentic applications; Foundry guardrails at input, tool-call, tool-response and output intervention points; securing agents with least agency, per-agent identity, approvals, sandboxing and egress control; Purview data protection and DSPM for AI; Defender for Cloud AI posture and threat protection; data privacy, residency and abuse monitoring commitments; the EU AI Act (including the 2026 AI Omnibus changes), NIST AI RMF and ISO/IEC 42001; an AI governance operating model; and governing agents at scale with Entra Agent ID, Foundry Control Plane and Agent 365.

## What you will be able to do

- Apply Microsoft's responsible AI principles through impact assessments and the Discover, Protect, Govern lifecycle.
- Threat-model AI apps and agents against the OWASP LLM and agentic Top 10.
- Configure and deploy guardrails with the right risks, intervention points and blocking levels.
- Design agent security with least privilege, approvals, sandboxing, egress control and a kill switch.
- Protect data used by AI with Purview labels, DLP, DSPM and oversharing controls.
- Enable and operate Defender for Cloud AI posture management and threat protection.
- Explain precisely how Azure handles prompts and completions, residency by deployment type, and abuse monitoring.
- Classify use cases under the EU AI Act and map obligations to Azure controls.
- Design an AI governance operating model with tiered gates, inventory and policy enforcement.
- Govern an agent fleet across Entra Agent ID, Foundry Control Plane and Agent 365.

## Prerequisites

[Topic 02](../02-identity-and-access/README.md) (identity, especially file 10), [Topic 03](../03-governance-and-landing-zones/README.md) (policy, landing zones, compliance), [Topic 18](../18-agents-with-foundry-agent-service/README.md) (agents) and [Topic 22](../22-evaluation-observability-and-genaiops/README.md) (evaluation, red teaming).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Responsible AI Principles](01-responsible-ai-principles.md) | 35 min | Impact assessment |
| 02 | [AI Threat Landscape](02-ai-threat-landscape.md) | 45 min | Threat model with OWASP lists |
| 03 | [Guardrails and Content Filters](03-guardrails-and-content-filters.md) | 50 min | Guardrail in portal and Bicep |
| 04 | [Securing Agents](04-securing-agents.md) | 50 min | Approvals, agent guardrail, kill switch |
| 05 | [Data Protection for AI](05-data-protection-for-ai.md) | 45 min | DSPM, oversharing, DLP |
| 06 | [Defender for Cloud AI Protection](06-defender-for-cloud-ai-protection.md) | 40 min | Enable plans, user context, alerts |
| 07 | [Data Privacy, Residency and Abuse Monitoring](07-data-privacy-residency-and-abuse-monitoring.md) | 40 min | Data inventory, residency policy |
| 08 | [AI Regulation and Standards](08-ai-regulation-and-standards.md) | 45 min | EU AI Act classification |
| 09 | [AI Governance Operating Model](09-ai-governance-operating-model.md) | 45 min | Intake, tiering, policy |
| 10 | [Agent Governance at Scale](10-agent-governance-at-scale.md) | 40 min | Registry, Control Plane, KPIs |

**AI engineers:** 02–04 and 06–07 in depth, the rest for context. **Architects:** all of it. **Consultants:** 01, 02, 05, 07–10, with 03–04 at concept level.

## Completion checklist

- [ ] I have written an impact assessment for an AI use case and mapped harms to mitigations.
- [ ] I can explain indirect prompt injection and threat-model an agent against the OWASP agentic Top 10.
- [ ] I have deployed a guardrail with indirect attack detection and assigned an agent guardrail.
- [ ] I have implemented human approval for write tools and tested a kill-switch runbook.
- [ ] I have run a DSPM oversharing assessment and configured a DLP policy for AI.
- [ ] I have enabled Defender AI threat protection and seen an alert with user context.
- [ ] I can explain Azure's data commitments, residency per deployment type and abuse monitoring.
- [ ] I can classify use cases under the EU AI Act and map obligations to controls.
- [ ] I have designed an AI intake and tiering process with automated policy enforcement.
- [ ] I can explain how Entra Agent ID, Foundry Control Plane and Agent 365 work together.

## Maps to

- **AI-103**: implement responsible AI, content safety and secure AI solutions.
- **AZ-305**: design governance and security for AI workloads.
- **SC-100 / SC-500** (adjacent): security architecture and operations for AI.
- **AB-100**: agent governance and responsible AI for business solutions.

## Next

24 · [Copilot, Low-Code Agents and Azure AI Assistants](../24-copilot-and-low-code-agents/README.md)
