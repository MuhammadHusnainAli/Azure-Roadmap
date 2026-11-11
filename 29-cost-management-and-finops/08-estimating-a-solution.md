# Estimating a Solution

## Why it matters

Every proposal, business case and architecture review includes a cost estimate, and it's the number stakeholders remember. Estimates that are too low destroy trust when the bills arrive; estimates that are padded lose deals and approvals. A **defensible estimate** states its assumptions, covers every cost component (not just compute), shows scenarios, compares options, includes total cost of ownership (TCO) beyond Azure charges, and is validated against real usage as soon as possible. This file brings together pricing fundamentals (file 01), commitments (file 03), optimisation (file 05) and AI modelling (file 06) into a repeatable estimation method for architects and consultants.

## Key concepts

### Estimate quality levels

| Level | When | Accuracy (typical) | Basis |
|---|---|---|---|
| **Order of magnitude** | Early idea, RFP qualification | −50% to +100% | Analogies, reference architectures |
| **Budgetary** | Business case, architecture options | −25% to +50% | Pricing calculator with documented assumptions |
| **Definitive** | Final proposal, budget commitment | −10% to +15% | Detailed design, measured workloads (Azure Migrate, load tests, telemetry) |

State which level you're giving.

### Components to include

| Category | Items |
|---|---|
| **Compute** | VMs, App Service, Container Apps, AKS nodes, Functions, batch |
| **Data** | Databases, storage (capacity, transactions, redundancy), backups, replicas |
| **Networking** | Egress, inter-region, private endpoints, firewalls, gateways, Front Door, DDoS |
| **AI** | Model tokens/PTU, embeddings, search, agent runtime, evaluation (file 06) |
| **Security and governance** | Defender plans, Sentinel, Key Vault, Purview |
| **Operations** | Log Analytics, Application Insights, backup, Site Recovery, automation |
| **Environments** | Dev, test, staging, DR (often 30–60% of production cost combined) |
| **Licences** | Windows/SQL (or AHB), third-party, Marketplace, Microsoft 365 Copilot/Agent 365 where relevant |
| **Support** | Azure support plan |
| **People and services** | Migration/build effort, operations team, training (for TCO) |
| **One-off** | Migration tooling, data transfer, parallel running |

### TCO

TCO compares the full cost of the current state (on-premises hardware, datacentre, power, licences, staff, refresh cycles) with the target state over 3–5 years. Tools: **Azure TCO calculator** and **Azure Migrate business case** (from discovered inventory and utilisation, Topic 30). Include benefits (avoided refresh, reduced downtime) separately from costs; be explicit about soft savings.

### Scenarios and sensitivity

Present **low/expected/high** usage scenarios and identify the 2–3 drivers that dominate (for example user growth, tokens per conversation, data volume). Show the effect of commitments (PAYG vs 1-year vs 3-year) and optimisation levers.

### Assumptions register

Every estimate needs a list of assumptions: regions, hours of operation, growth rates, data volumes, redundancy choices, discounts applied (EA/MCA rates, AHB, reservations), exchange rate and price date. Assumptions are what make an estimate defensible and reviewable.

## How it works on Azure

Estimation workflow:

1. **Gather requirements**: workload profile (users, transactions, data, growth), SLOs (Topic 28), compliance (regions), environments.
2. **Design options**: for example PaaS vs IaaS, single-region vs multi-region, PAYG vs PTU.
3. **Size** each component from measurements (Azure Migrate assessment, current monitoring, load tests) or benchmarks.
4. **Price** in the pricing calculator (or Retail Prices API for automation), applying negotiated rates where available; export the estimate.
5. **Add** non-Azure and people costs for TCO.
6. **Model** scenarios and commitments in a spreadsheet; show monthly and 3-year totals.
7. **Review** with a peer and the customer's finance contact; record assumptions.
8. **Validate** after deployment: compare Cost Management actuals by tag with the estimate in month 1–3; adjust.

Example summary table for a proposal:

| Component | Expected monthly (PAYG) | With 3-year commitments | Notes |
|---|---|---|---|
| App tier (Container Apps, 2 regions active-passive) | €2,400 | €1,700 | Compute savings plan |
| Azure SQL Hyperscale + failover group | €3,100 | €2,300 | Database savings plan |
| AI (tokens, AI Search S1 ×2 replicas) | €2,000 | €2,000 | PAYG until usage stabilises |
| Networking (Front Door Premium, firewall, egress) | €1,800 | €1,800 | Egress assumed 5 TB/month |
| Security and monitoring | €1,500 | €1,400 | Defender CSPM, servers P2, 150 GB/day logs |
| Non-production environments | €2,500 | €2,000 | Auto-shutdown outside hours |
| **Total** | **€13,300** | **€11,200** | Prices as of Dec 2026, EUR, list prices |

Automating estimates with IaC: tools can price Bicep/Terraform plans in pull requests (for example open-source cost estimation tools), making cost visible during design.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Precision vs effort** | Match the estimate level to the decision being made. |
| **List vs negotiated prices** | Show list prices unless authorised to use customer-specific rates; state which. |
| **Commitments in estimates** | Show PAYG and committed scenarios; don't assume commitments for uncertain workloads. |
| **Contingency** | Add explicit contingency for uncertainty instead of hiding padding. |
| **Ownership** | Hand the estimate and assumptions to the FinOps owner for tracking after go-live. |
| **AI uncertainty** | Wider ranges for AI usage; plan a measured pilot first. |

## Hands-on

1. Build a budgetary estimate for a three-tier web app with AI search using the pricing calculator; export it.
2. Write the assumptions register (at least ten assumptions).
3. Create low/expected/high scenarios and identify the top three cost drivers.
4. Run the TCO calculator for a 50-server on-premises estate and compare with an Azure Migrate business case if available.
5. After deploying a test environment, compare a month's actual cost by tag with your estimate and explain variances.

## Security, cost and reliability notes

- **Don't cut security or resilience to win on price** without documenting the risk.
- **Customer pricing is confidential**: handle EA/MCA rates carefully in proposals.
- **Reliability costs**: zone redundancy, DR and backups must be in the estimate if they're in the design.

## Common mistakes

| Mistake | Fix |
|---|---|
| Compute-only estimates | Include data, network, security, operations, environments |
| No assumptions | Assumptions register |
| Single number with false precision | Ranges and scenarios |
| Ignoring non-production and DR | Include all environments |
| Never checking estimate vs actual | Post-deployment validation |

## Check yourself

1. What are the three estimate levels and their typical accuracy?
2. List ten cost components often missed.
3. What does TCO include, and which tools help?
4. Why use scenarios and sensitivity analysis?
5. What belongs in an assumptions register?
6. How do you validate an estimate after go-live?

<details>
<summary>Answers</summary>

1. Order of magnitude (−50% to +100%), budgetary (−25% to +50%), definitive (−10% to +15%).
2. Any ten: egress, inter-region transfer, transactions, backups, replicas, logs/App Insights, Defender/Sentinel, Key Vault, non-production environments, DR, licences, support, people, migration one-offs, AI evaluation/search.
3. Full current and target costs over 3–5 years including hardware, facilities, licences and staff; Azure TCO calculator and Azure Migrate business case.
4. Usage is uncertain; they show ranges and which drivers matter most.
5. Regions, hours, growth, volumes, redundancy, discounts, exchange rates, price date and other inputs.
6. Compare Cost Management actuals by tag with the estimate in the first months and explain/adjust variances.

</details>

## References

- [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/)
- [Azure TCO calculator](https://azure.microsoft.com/pricing/tco/calculator/)
- [Azure Migrate business case](https://learn.microsoft.com/azure/migrate/concepts-business-case-calculation)
- [Cost efficiency (CAF strategy)](https://learn.microsoft.com/azure/cloud-adoption-framework/strategy/inform/cost-efficiency)
- [Well-Architected cost optimisation](https://learn.microsoft.com/azure/well-architected/cost-optimization/)
