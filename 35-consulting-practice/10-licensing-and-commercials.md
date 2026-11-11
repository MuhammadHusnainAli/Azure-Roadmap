# Licensing and Commercials

## Why it matters

Clients ask consultants "how do we buy this, and what will it cost?" as often as "how do we build it?". Microsoft's commercial landscape is complex: Azure agreements (Microsoft Customer Agreement, Enterprise Agreement, Cloud Solution Provider), consumption commitments (MACC), reservations and savings plans, Microsoft 365 Copilot licences and usage-based billing, Copilot Studio capacity, Agent 365 per-user licensing, Security Copilot capacity, marketplace purchases, and partner programmes and funding. Getting commercials right can change a business case significantly; getting them wrong creates surprise bills or compliance issues. Consultants don't replace licensing specialists, but they must understand the models well enough to design cost-effective solutions and ask the right questions.

## Key concepts

### Azure purchasing agreements

| Agreement | Description | Notes |
|---|---|---|
| **Microsoft Customer Agreement (MCA)** | Modern agreement; billing account → billing profiles → invoice sections | MCA-E (enterprise, via Microsoft), MCA online; increasingly the default |
| **Enterprise Agreement (EA)** | Traditional enterprise agreement with enrolment hierarchy | Being transitioned to MCA for many customers |
| **Cloud Solution Provider (CSP)** | Purchase through a partner who bills and supports | Partner-managed relationship and support |
| **Pay-as-you-go** | Credit card/invoice subscriptions | Small or initial use |
| **MACC** | Commitment to spend an amount over a term in exchange for discounts/benefits | Eligible Marketplace purchases count (Topic 32 file 08) |

### Azure discounts and benefits (Topic 29 file 03)

Reservations, savings plans (compute and database), Azure Hybrid Benefit, dev/test pricing, Extended Security Updates via Arc, negotiated discounts, and credits from programmes.

### Copilot and agent licensing (check current terms)

| Product | Licensing model (at time of writing) |
|---|---|
| **Microsoft 365 Copilot Chat (Basic)** | Included with eligible Microsoft 365 licences; web grounding; pay-as-you-go agents for work data |
| **Microsoft 365 Copilot** | Per-user add-on (Premium experience); usage-based billing options for Copilot Cowork and Copilot Code |
| **Microsoft 365 E7** | Includes Microsoft 365 E5, Microsoft Copilot, Agent 365 and Entra Suite |
| **Copilot Studio** | Billing depends on harness: usage-based billing/Copilot credits; capacity packs or pay-as-you-go for standard harness agents; use rights included with some Microsoft 365 Copilot licences |
| **Microsoft Agent 365** | Per-user licensing; works best with E5; required for Defender agent discovery/posture (from 1 July 2026) |
| **Security Copilot** | Security Compute Units (SCUs); included allocation with Microsoft 365 E5/E7 (400 SCUs per 1,000 paid licences per month, up to 10,000) plus purchasable capacity |
| **GitHub Copilot** | Per-user plans; AI credits and compute allowances for cloud agent experiences |
| **Foundry / Azure OpenAI** | Consumption (tokens), provisioned throughput (PTU hourly/reserved), Azure services billed separately |
| **Power Platform / Dynamics 365** | Per-user/per-app licences, AI Builder/Copilot credits |

Licensing names, bundles and entitlements change frequently; confirm with Microsoft licensing guides or a licensing specialist before advising.

### Partner programmes and funding

| Programme | Notes |
|---|---|
| **Microsoft AI Cloud Partner Program** | Partner membership, solutions partner designations and specialisations; benefits and eligibility for incentives |
| **Customer investment programmes** | Microsoft provides funding, deployment assistance and expert help for qualifying Azure, migration and AI projects (for example Frontier Accelerate for Azure and Cloud Accelerate Factory at the time of writing) |
| **Marketplace** | Co-sell, MACC-eligible transactable offers |

Engage the Microsoft account team early: programmes, funding and licensing guidance can materially change economics.

## How it works on Azure

Commercial checklist for a solution proposal:

| Question | Why |
|---|---|
| Which agreement does the client have (MCA/EA/CSP)? | Billing structure, discounts, purchasing path |
| Is there a MACC, and how much is unconsumed? | Align spend and marketplace choices |
| Existing licences (Windows Server, SQL with SA, Microsoft 365 E3/E5/E7, Copilot)? | AHB, included capabilities, avoiding duplicate purchases |
| Commitment opportunities (reservations, savings plans, PTU reservations)? | Rate optimisation |
| Copilot/agent licensing needs per user group? | Business case accuracy |
| Eligible Microsoft programmes or funding? | Lower client investment |
| Marketplace third-party components? | MACC eligibility, procurement |

Use Cost Management at billing scopes for current spend and commitments; licensing portals (Microsoft 365 admin centre) for seat counts and usage.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Included vs paid capabilities** | Maximise what's already licensed (for example E5 security, Copilot Chat) before buying more. |
| **Per-user vs consumption** | Per-user for heavy, predictable users; consumption for occasional use. |
| **Commit vs flexibility** | Commit for stable baselines; stay flexible for new AI workloads. |
| **Independence** | Advise in the client's interest; disclose partner incentives. |
| **Specialist involvement** | Use licensing specialists for complex agreements and audits. |

## Hands-on

1. Map a client's agreements and licences into a commercial profile.
2. Compare per-user vs consumption costs for an agent used by 500 occasional and 50 heavy users.
3. Identify included capabilities (for example E5 security, Security Copilot allocation) that reduce new purchases.
4. List Microsoft programmes the client might qualify for and how to engage.
5. Add a commercial section to a proposal with assumptions and disclaimers.

## Security, cost and reliability notes

- **Licence compliance** risks: AHB eligibility, Copilot assignments, Software Assurance terms; audits can be costly.
- **Usage-based AI** needs budgets and limits to avoid bill shock.
- **Commercial data** (discounts, agreements) is confidential.

## Common mistakes

| Mistake | Fix |
|---|---|
| Ignoring existing entitlements | Licence inventory first |
| Assuming list prices | Use the client's agreement pricing |
| Over-buying Copilot licences | Pilot and assign by use case |
| Forgetting programme funding | Engage Microsoft account team |
| Giving definitive licensing advice without verification | Confirm with specialists |

## Check yourself

1. Compare MCA, EA and CSP.
2. What is a MACC?
3. Summarise Copilot, Copilot Studio, Agent 365 and Security Copilot licensing models.
4. What's included in Microsoft 365 E7?
5. Which questions belong on a commercial checklist?
6. Why engage the Microsoft account team early?

<details>
<summary>Answers</summary>

1. MCA: modern agreement with billing profiles/invoice sections; EA: traditional enrolment-based agreement; CSP: purchase and support via a partner.
2. A contractual commitment to spend a set amount on Azure over a term, with eligible Marketplace purchases counting.
3. Copilot: per-user add-on plus usage-based billing options; Copilot Studio: harness-dependent usage/credits or capacity; Agent 365: per-user; Security Copilot: SCUs with E5/E7 included allocation.
4. Microsoft 365 E5, Microsoft Copilot, Microsoft Agent 365 and Microsoft Entra Suite.
5. Agreement type, MACC status, existing licences, commitment opportunities, Copilot/agent licensing needs, programmes/funding, marketplace components.
6. For programmes, funding, licensing guidance and product expertise that can change the economics and speed.

</details>

## References

- [Microsoft Customer Agreement overview](https://learn.microsoft.com/azure/cost-management-billing/understand/mca-overview)
- [Enterprise Agreement](https://learn.microsoft.com/azure/cost-management-billing/manage/ea-portal-agreements)
- [Cloud Solution Provider overview](https://learn.microsoft.com/partner-center/csp-overview)
- [Microsoft 365 Copilot licensing](https://learn.microsoft.com/copilot/microsoft-365/microsoft-365-copilot-licensing)
- [Copilot Studio billing and licensing](https://learn.microsoft.com/microsoft-copilot-studio/billing-licensing)
- [Microsoft Agent 365 overview](https://learn.microsoft.com/microsoft-agent-365/overview)
- [Security Copilot inclusion in Microsoft 365 E5](https://learn.microsoft.com/copilot/security/security-copilot-inclusion)
