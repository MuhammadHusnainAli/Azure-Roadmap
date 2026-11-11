# Client Engagement Simulation

## Why it matters

Consulting skills grow through practice, not reading. This capstone simulates a full early-stage engagement. You start from messy discovery notes and produce a prioritised use case, a proposal with estimate, a high-level design document and an executive readout. It's the core portfolio piece for the **Azure Consultant** track and a valuable exercise for architects who want to lead engagements.

## Key concepts

### The client: Margate Logistics (fictional)

**Discovery notes (as captured, unedited):**

```text
- 3,200 employees, UK + Netherlands. Warehouses x14. Customer service team 120 FTE.
- CEO wants "an AI strategy" by Q2; board asked about competitors using AI.
- CS team handles ~45k emails/month: delivery status (≈50%), damage claims (≈15%), rebooking (≈20%), other.
- Avg handling time 9 min/email; backlog spikes at peak season; CSAT dropped from 78 to 71.
- Systems: Dynamics 365 Customer Service (2 yrs), SAP for finance, custom TMS (tracking) with REST API, SharePoint for SOPs.
- Microsoft 365 E3; 50 Copilot licences in pilot, usage "patchy".
- Azure: a few subscriptions, no landing zone, IT team of 25, 2 people know Azure.
- Data: tracking data good; SOPs outdated in places; claims process has paper forms.
- Concerns: GDPR (NL works council must be consulted), "AI making promises to customers", cost.
- Budget signal: "Six figures for year one if it pays back."
- Champion: Head of Customer Service. Sceptic: CFO. IT Director worried about support burden.
```

### Deliverables

| # | Deliverable | Guidance |
|---|---|---|
| D1 | Use-case backlog and prioritisation | At least six use cases scored on value, feasibility and risk ([35 file 03](../35-consulting-practice/03-use-case-prioritisation-and-business-case.md)) |
| D2 | Business case for the top use case | Three scenarios, assumptions stated, benefits tied to handling time, CSAT and backlog |
| D3 | Readiness summary | Data, platform, skills, governance gaps ([35 file 04](../35-consulting-practice/04-ai-readiness-assessment.md)) |
| D4 | Proposal and statement of work | Phased plan (foundation, pilot, scale), PERT estimate, assumptions, client responsibilities, exit criteria ([35 file 06](../35-consulting-practice/06-proposals-estimation-and-statements-of-work.md)) |
| D5 | High-level design | Architecture, build-surface choice, security, responsible AI, operations, cost ([35 file 07](../35-consulting-practice/07-solution-design-deliverables.md)) |
| D6 | Executive readout | 10 slides, situation–complication–resolution, decision requested ([35 file 08](../35-consulting-practice/08-stakeholder-communication.md)) |
| D7 | Adoption and governance plan | Champions, training, works council engagement, RAID log ([35 files 09](../35-consulting-practice/09-change-management-and-adoption.md) and [11](../35-consulting-practice/11-delivery-governance.md)) |

### Skills practised

All of [Topic 35](../35-consulting-practice/README.md), plus [24](../24-copilot-and-low-code-agents/README.md), [34](../34-ai-solution-architecture/README.md) and [03 file 04](../03-governance-and-landing-zones/04-ai-landing-zones.md).

## How it works on Azure

### A plausible top use case (don't read until you've done D1)

<details>
<summary>Example direction</summary>

An **email triage and draft-response assistant** for delivery status and rebooking:
- Integrated with Dynamics 365 Customer Service (Copilot in Customer Service and/or a Copilot Studio or Foundry agent).
- Grounded in TMS tracking data via an API tool and in SOPs via retrieval.
- Agents review drafts before sending (human in the loop) in phase 1.
- Automated sending only for low-risk status replies in phase 2, after evaluation proves quality.

The foundation work runs alongside it:
- An AI-ready landing zone.
- An SOP clean-up.
- A works council consultation and a DPIA.
- An adoption plan for the 50 Copilot licences.

</details>

### Estimating guidance

| Phase | Typical content |
|---|---|
| 0 Mobilise and discovery validation | Stakeholder interviews, data access, success metrics |
| 1 Foundation | Landing zone, identity, networking, governance for AI |
| 2 Pilot | One use case, one team (~20 agents), evaluation, adoption, measured outcomes |
| 3 Scale decision | Business case refresh with real data; go or no-go for scale |

Use three-point estimates per work package and add contingency for data and integration uncertainty.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Build surface | Dynamics 365 Customer Service with Copilot Studio vs a custom Foundry agent: weigh existing investment, skills, control and cost |
| Automation level | Draft-and-review first; automate only proven low-risk categories |
| Scope vs budget | Fit phase 1–2 within the "six figures" signal; show the scale cost separately |
| Stakeholders | CFO needs measurable payback; IT Director needs a support model; works council needs transparency |

## Hands-on

1. Produce D1–D7 within a time box (for example 3 days of effort).
2. Present the executive readout to a peer playing the CFO and handle objections.
3. Have a second peer review the SoW for ambiguity and missing assumptions.
4. Reflect: which assumptions carry the most risk, and how would you test them early?
5. Package the deliverables (anonymised, fictional) in your portfolio.

## Security, cost and reliability notes

- GDPR and the Netherlands works council: plan a DPIA and consultation early, as they can gate the timeline.
- "AI making promises": use human review, grounded responses, approved templates and guardrails.
- Cost: include licensing (Copilot, Copilot Studio or Dynamics 365 capacity), Azure consumption and support.
- Support burden: define the operating model and handover for a small IT team.

## Common mistakes

| Mistake | Fix |
|---|---|
| Jumping to a solution before prioritising | Score use cases first |
| Business case with unsupported benefits | Tie benefits to measured baselines |
| Ignoring the 50 underused Copilot licences | Quick-win adoption plan |
| SoW without assumptions or client responsibilities | Make them explicit |
| A technical readout for executives | Outcome-first storytelling |

## Check yourself

1. Which data points in the notes support a business case?
2. Which stakeholders must you win over, and how?
3. Why start with draft-and-review rather than automated replies?
4. Which foundation work is needed before a pilot?
5. What must the SoW make explicit?

<details>
<summary>Answers</summary>

1. 45,000 emails a month, the category split, 9 minutes average handling time, backlog spikes and the CSAT drop.
2. The CFO (payback evidence), the IT Director (support model and platform) and the works council (transparency and consultation), with the Head of Customer Service as champion.
3. To manage the risk of incorrect promises, build trust and gather evaluation data before automating.
4. AI-ready landing zone, identity and governance, data access to TMS and SOPs, a DPIA and the works council consultation.
5. Scope, deliverables, assumptions, client responsibilities, exit criteria, change control and commercial terms.

</details>

## References

- [Cloud Adoption Framework: AI adoption](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/)
- [Configure Copilot in Dynamics 365 Customer Service](https://learn.microsoft.com/dynamics365/customer-service/administer/configure-copilot-features)
- [Copilot Studio overview](https://learn.microsoft.com/microsoft-copilot-studio/fundamentals-what-is-copilot-studio)
- [Microsoft Adoption hub](https://adoption.microsoft.com/)
