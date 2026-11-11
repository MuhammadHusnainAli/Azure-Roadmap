# Microsoft Marketplace and Partner Solutions

## Why it matters

No organisation builds everything itself. Security tools, observability platforms, databases, industry applications and, increasingly, AI apps and agents come from software partners. **Microsoft Marketplace** (the unified successor to Azure Marketplace and AppSource) is where customers discover, buy and deploy partner solutions, including **AI apps and agents**, with purchases billed through Microsoft and, for eligible offers, counting towards the customer's **Microsoft Azure Consumption Commitment (MACC)**. **Azure Native Integrations** (formerly Azure Native ISV Services) go further, embedding partner services such as observability and data platforms directly into Azure with Entra SSO, Azure Monitor integration and consolidated billing. Consultants and architects regularly recommend, procure or publish through these channels.

## Key concepts

### Microsoft Marketplace for customers

| Capability | Detail |
|---|---|
| **Discovery** | Thousands of solutions, including AI apps and agents; badges, ratings and reviews; Azure benefit-eligible filters |
| **Offer types** | SaaS subscriptions, Azure applications (managed applications and solution templates), virtual machine images, containers, consulting services, AI agents/apps for Microsoft 365 and Copilot |
| **Purchasing** | Trials, pay-as-you-go, subscriptions; required billing roles and permissions |
| **Private offers** | Custom pricing and terms negotiated with a publisher, accepted in the marketplace |
| **Private Azure Marketplace / governance** | Admins restrict which offers users can purchase |
| **MACC benefit** | Eligible ("Azure benefit eligible") purchases count towards the Azure consumption commitment |
| **Billing** | Consolidated on the Microsoft invoice |

### Azure Native Integrations

Jointly developed by Microsoft and partners, managed as Azure resources:

- integrated onboarding via ARM, SDKs, CLI and portal;
- automatic resource discovery and integration of logs and metrics with Azure Monitor;
- Entra SSO;
- consolidated billing through the marketplace;
- private data plane access and VNet injection where supported;
- solution areas: AI, observability (for example Datadog, Dynatrace, Elastic, New Relic), data and storage, and others.

### For publishers (partners)

| Topic | Detail |
|---|---|
| **Partner Center** | Create and manage offers |
| **Transactable offers** | Microsoft handles billing; agency fee applies |
| **Co-sell / Azure IP co-sell** | Joint selling with Microsoft; MACC eligibility requires Azure IP co-sell requirements |
| **AI apps and agents** | Publish SaaS or Azure-hosted AI apps and agents, including those integrating with Microsoft 365 Copilot and Agent 365 |
| **Managed applications** | Deploy into customer subscriptions with publisher management access |

## How it works on Azure

Procurement pattern for an enterprise:

1. **Govern**: enable private marketplace; curate approved offers; set purchasing permissions.
2. **Evaluate**: security review, data residency, integration, support, licensing; check Azure benefit eligibility.
3. **Negotiate**: private offer for enterprise pricing; ensure MACC eligibility if relevant.
4. **Purchase and deploy**: subscribe (SaaS) or deploy (Azure application) into a governed subscription; for Azure Native Integrations, create the resource and connect Entra SSO and Monitor.
5. **Operate**: monitor usage and cost in Cost Management (marketplace charges appear separately); renewals and licence management.

Example: deploy an Azure Native observability integration: create the partner resource in the portal, choose subscriptions to monitor, enable automatic log and metric forwarding, sign in through Entra SSO, and review costs on the Azure invoice.

AI agent procurement considerations: data handling (does the agent send data outside your tenant?), identity (Entra Agent ID, permissions requested), governance (Agent 365 registry), security assessment, and alignment with responsible AI policies (Topic 23).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Marketplace vs direct purchase** | Marketplace for consolidated billing, MACC burn-down and procurement speed; direct when terms or support require it. |
| **Azure Native Integrations vs self-managed** | Native integrations for operational simplicity and Azure-integrated billing/SSO. |
| **Private offers** | Use for enterprise discounts and custom terms. |
| **Governance** | Private marketplace and purchase permissions prevent shadow procurement. |
| **Third-party AI agents** | Apply the same security, data and governance review as internal agents. |

## Hands-on

1. Browse Microsoft Marketplace for an AI agent and an observability solution; note eligibility for Azure benefits.
2. Enable a private marketplace and approve two offers in a test tenant.
3. Deploy an Azure Native Integration (trial) and connect it to Azure Monitor and Entra SSO.
4. Write a third-party AI agent assessment checklist.
5. Check how marketplace charges appear in Cost Management.

## Security, cost and reliability notes

- **Data flows to partners**: review where data goes and under which terms.
- **Managed applications** grant publishers access to resources in your subscription; review permissions.
- **Cost**: marketplace charges outside Azure commitments unless eligible; track renewals.
- **Vendor risk**: include marketplace vendors in third-party risk management.

## Common mistakes

| Mistake | Fix |
|---|---|
| Uncontrolled marketplace purchases | Private marketplace and permissions |
| Assuming every purchase counts towards MACC | Check benefit eligibility |
| Skipping security review for SaaS AI agents | Same assessment as internal agents |
| Ignoring managed application access | Review publisher permissions |
| Losing track of renewals | Licence management |

## Check yourself

1. What is Microsoft Marketplace, and what did it unify?
2. Name four offer types.
3. What are private offers and private marketplace?
4. What do Azure Native Integrations provide?
5. When do marketplace purchases count towards MACC?
6. What should you check before buying a third-party AI agent?

<details>
<summary>Answers</summary>

1. Microsoft's marketplace for discovering, buying and deploying partner solutions including AI apps and agents; it unified Azure Marketplace and AppSource.
2. Any four: SaaS, Azure applications (managed apps/solution templates), VM images, containers, consulting services, AI apps/agents.
3. Private offers: negotiated custom pricing and terms; private marketplace: admin-curated list of allowed offers.
4. Partner services integrated into Azure with ARM onboarding, Monitor integration, Entra SSO, consolidated billing and private networking options.
5. When the offer is Azure benefit eligible (meets Azure IP co-sell requirements).
6. Data handling and residency, identity and permissions, governance registration, security assessment, responsible AI alignment, support and pricing.

</details>

## References

- [Microsoft Marketplace customer documentation](https://learn.microsoft.com/marketplace/)
- [Private offers](https://learn.microsoft.com/marketplace/private-offers-overview)
- [Azure consumption commitment benefit](https://learn.microsoft.com/marketplace/azure-consumption-commitment-benefit)
- [Azure Native Integrations](https://learn.microsoft.com/azure/partner-solutions/overview)
- [Marketplace offers for publishers](https://learn.microsoft.com/partner-center/marketplace-offers/overview)
