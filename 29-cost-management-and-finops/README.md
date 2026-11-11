# 29 · Cost Management and FinOps

> Tracks: E ○ A ● C ● · Part E · Secure, Operate, Migrate and Extend

Cloud spend is a design output, not an accident. This topic covers how Azure prices and bills services (meters, billing scopes, actual vs amortised cost); Microsoft Cost Management for analysis, budgets, anomaly alerts and FOCUS exports; commitment discounts including reservations, savings plans for compute and for databases, Azure Hybrid Benefit and MACC; cost allocation with tagging, showback, chargeback and shared-cost rules; usage optimisation techniques; AI cost modelling with token economics and PTU break-even; the FinOps Framework and Microsoft's FinOps toolkit; and building defensible estimates and TCO for proposals.

## What you will be able to do

- Explain Azure pricing models, meters, billing scopes and invoices.
- Use Cost Management to analyse spend, set budgets and detect anomalies.
- Choose and size reservations, savings plans and licensing benefits.
- Design tagging and allocation so every cost has an owner.
- Find and remove waste and rightsize resources safely.
- Model AI solution costs and decide between PTU and pay-as-you-go.
- Stand up a FinOps practice with KPIs and the FinOps toolkit.
- Produce estimates and TCO with assumptions and scenarios.

## Prerequisites

[Topic 03](../03-governance-and-landing-zones/README.md) (management groups, policy, tagging), [Topic 15](../15-foundry-models-and-azure-openai/README.md) for AI pricing concepts; basic spreadsheet skills.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Azure Pricing Fundamentals](01-azure-pricing-fundamentals.md) | 35 min | Calculator, Retail Prices API |
| 02 | [Microsoft Cost Management](02-microsoft-cost-management.md) | 40 min | Views, budgets, FOCUS export |
| 03 | [Commitment Discounts](03-commitment-discounts.md) | 40 min | Recommendations, break-even |
| 04 | [Cost Allocation](04-cost-allocation.md) | 35 min | Tag policies, allocation rules |
| 05 | [Optimisation Techniques](05-optimisation-techniques.md) | 40 min | Waste queries, lifecycle, spot |
| 06 | [AI Cost Modelling](06-ai-cost-modelling.md) | 45 min | Token model, PTU break-even |
| 07 | [FinOps Framework](07-finops-framework.md) | 35 min | Maturity assessment, FinOps hubs |
| 08 | [Estimating a Solution](08-estimating-a-solution.md) | 40 min | Estimate with assumptions and scenarios |

**Architects and consultants:** all of it. **AI engineers:** 01, 02, 05 and 06.

## Completion checklist

- [ ] I can read a Cost Management view at billing and resource scopes and explain actual vs amortised cost.
- [ ] I have budgets with forecast alerts and anomaly alerts routed to owners.
- [ ] I can recommend reservations vs savings plans vs AHB for a workload.
- [ ] I have enforced a tagging strategy and produced a showback report.
- [ ] I have removed waste and applied rightsizing recommendations.
- [ ] I have built an AI token cost model and validated it with telemetry.
- [ ] I have assessed FinOps maturity and deployed FinOps hubs or equivalent reporting.
- [ ] I have produced an estimate with assumptions, scenarios and TCO.

## Maps to

- **AZ-305**: design cost-optimised solutions.
- **AZ-104**: manage costs and budgets.
- **AB-100** (adjacent): business cases for AI solutions.

## Next

30 · [Migration and Modernisation](../30-migration-and-modernisation/README.md)
