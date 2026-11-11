# FinOps Framework

## Why it matters

Tools alone don't control cloud costs. Engineers make spending decisions every day (instance sizes, scaling rules, model choices, log levels), finance needs predictable forecasts, and leadership wants to know whether cloud spend creates value. **FinOps** is the operating model that brings these groups together: a cultural practice and framework for data-driven, timely decisions that maximise the business value of cloud. Microsoft is a member of the FinOps Foundation (since February 2023), aligns its guidance with the **FinOps Framework**, and publishes the open-source **FinOps toolkit** (FinOps hubs, Power BI reports, workbooks, alerts, Optimization Engine, PowerShell and open data). Consultants are often asked to assess FinOps maturity and stand up a practice.

## Key concepts

### Principles (FinOps Foundation)

Teams collaborate; business value drives technology decisions; everyone takes ownership of their usage; FinOps data is accessible, timely and accurate; FinOps is enabled centrally; take advantage of the variable cost model of the cloud.

### Lifecycle phases

| Phase | Focus | Examples |
|---|---|---|
| **Inform** | Visibility and shared accountability | Allocation, showback, budgets, forecasts, benchmarks, anomaly detection |
| **Optimise** | Reduce waste, improve efficiency | Rightsizing, scheduling, commitment discounts, architecture changes |
| **Operate** | Governance and continuous improvement | KPIs, policies, automation, education, decision processes |

Teams iterate through phases continuously, at different maturity per capability.

### Domains and capabilities (examples)

| Domain | Capabilities |
|---|---|
| **Understand usage and cost** | Data ingestion, allocation, reporting and analytics, anomaly management |
| **Quantify business value** | Planning and estimating, forecasting, budgeting, benchmarking, unit economics |
| **Optimise usage and cost** | Architecting for cloud, workload optimisation, rate optimisation, licensing, sustainability |
| **Manage the FinOps practice** | Education and enablement, policy and governance, invoicing and chargeback, tools and services, intersecting disciplines |

### Maturity

The framework's simple **Crawl–Walk–Run** model is applied **per capability**, not globally: you might be "Run" at reporting and "Crawl" at forecasting. Assess, pick a few capabilities to improve, iterate.

### Personas

FinOps practitioners, engineering and product teams, finance and procurement, leadership, plus intersecting disciplines (ITAM, ITSM, ITFM/TBM, sustainability, security).

### FinOps toolkit (Microsoft, open source)

| Tool | Purpose |
|---|---|
| **FinOps hubs** | Data platform (Storage/Data Explorer or Fabric) ingesting Cost Management exports (FOCUS) at scale, with private networking support |
| **Power BI reports** | Starter kits for cost summary, rate optimisation, governance, data ingestion |
| **FinOps workbooks** | Azure Monitor workbooks for optimisation and governance |
| **FinOps alerts** | Automated notifications (for example idle resources) |
| **Azure Optimization Engine** | Extended recommendations beyond Advisor |
| **PowerShell module** | Automate exports, hubs and more |
| **Open data** | Reference datasets (pricing units, regions, services, resource types, instance size flexibility) |

### KPIs

| KPI | Purpose |
|---|---|
| % spend allocated | Allocation quality |
| Forecast accuracy (variance) | Planning quality |
| Commitment coverage and utilisation | Rate optimisation |
| Waste / idle cost | Usage optimisation |
| Unit cost (per customer, transaction, conversation) | Value and efficiency |
| Time to act on anomalies | Operational responsiveness |
| Savings realised | Programme outcomes |

## How it works on Azure

Standing up a FinOps practice (first 90 days):

1. **Sponsor and team**: executive sponsor, a small central FinOps team, champions in engineering and finance.
2. **Inform**: management group cost views, mandatory tags and inheritance (file 04), budgets and anomaly alerts (file 02), FOCUS exports into FinOps hubs, Power BI showback.
3. **Optimise**: Advisor and Optimization Engine backlog; rightsizing and scheduling quick wins (file 05); commitment strategy (file 03).
4. **Operate**: monthly FinOps review per business unit, KPIs, policy guardrails (allowed SKUs, regions, tags), cost included in architecture reviews and pull requests for IaC (cost estimates).
5. **Iterate**: maturity assessment per capability; choose next capabilities (forecasting, unit economics, AI cost management).

Deploy FinOps hubs (PowerShell module from the toolkit; check the current docs for parameters):

```powershell
Install-Module -Name FinOpsToolkit
Deploy-FinOpsHub -Name finops-hub-contoso -ResourceGroupName rg-finops -Location westeurope `
  -DataExplorerName adx-finops-contoso      # optional; recommended for large cost volumes
```

Then configure Cost Management exports (FOCUS) to the hub's storage and connect the Power BI reports.

FinOps for AI: track tokens and cost per AI workload and consumer via gateway metrics and exports; include AI in budgets, anomaly detection and unit economics (file 06).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Central vs federated** | Central team enables (data, tools, commitments, policy); teams own their spend and optimisation. |
| **Tooling** | Start with native Cost Management; add FinOps hubs/Power BI for scale, multi-cloud (FOCUS) and custom reporting. |
| **Cadence** | Weekly anomaly triage, monthly reviews, quarterly commitment and forecast reviews. |
| **Incentives** | Celebrate savings and efficient design; avoid blame. |
| **Policy vs culture** | Guardrails for hard limits; education and visibility for everything else. |
| **Sustainability** | Use Microsoft emissions data alongside cost where sustainability matters. |

## Hands-on

1. Assess maturity (Crawl/Walk/Run) for eight FinOps capabilities in your organisation.
2. Deploy FinOps hubs and connect a FOCUS export and the cost summary Power BI report.
3. Define five KPIs with targets and owners.
4. Run a monthly FinOps review for one business unit using real data.
5. Create a FinOps backlog combining Advisor, Optimization Engine and anomaly findings.

## Security, cost and reliability notes

- **Data access**: cost data can reveal pricing agreements; restrict access to hubs and reports.
- **Tool cost**: hubs (Data Explorer/Fabric) and Power BI have their own costs; size appropriately.
- **Don't trade reliability or security for savings** without explicit risk acceptance.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating FinOps as a finance-only function | Cross-functional with engineering ownership |
| One global maturity score | Per-capability maturity |
| Reports nobody acts on | Reviews with owners and backlogs |
| Ignoring unit economics | Measure cost per business outcome |
| Big-bang programmes | Iterative cycles |

## Check yourself

1. What is FinOps?
2. Describe the Inform, Optimise and Operate phases.
3. Name the four domains with examples of capabilities.
4. How is maturity assessed?
5. List five FinOps toolkit components.
6. Give five FinOps KPIs.

<details>
<summary>Answers</summary>

1. An operational framework and cultural practice that maximises business value from cloud through data-driven decisions and shared financial accountability across finance, technology and business.
2. Inform: visibility and accountability; Optimise: reduce waste and improve efficiency; Operate: governance, KPIs and continuous improvement.
3. Understand usage and cost (ingestion, allocation, reporting, anomalies); Quantify business value (estimating, forecasting, budgeting, benchmarking); Optimise usage and cost (architecture, workload and rate optimisation, licensing); Manage the FinOps practice (education, policy, chargeback, tools).
4. Crawl–Walk–Run per capability rather than one global score.
5. Any five: FinOps hubs, Power BI reports, workbooks, alerts, Optimization Engine, PowerShell module, open data.
6. Any five: % allocated, forecast accuracy, commitment coverage/utilisation, waste, unit cost, time to act on anomalies, savings realised.

</details>

## References

- [What is FinOps? (Microsoft)](https://learn.microsoft.com/cloud-computing/finops/overview)
- [FinOps Framework (Microsoft)](https://learn.microsoft.com/cloud-computing/finops/framework/finops-framework)
- [FinOps toolkit](https://microsoft.github.io/finops-toolkit/)
- [FinOps hubs](https://learn.microsoft.com/cloud-computing/finops/toolkit/hubs/finops-hubs-overview)
- [FinOps Foundation framework](https://www.finops.org/framework/)
