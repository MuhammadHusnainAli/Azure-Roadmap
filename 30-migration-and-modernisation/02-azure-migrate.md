# Azure Migrate

## Why it matters

You can't plan a migration from a spreadsheet someone updated two years ago. **Azure Migrate** is Microsoft's free hub for the migration journey: it **discovers** servers, databases and web apps (VMware, Hyper-V, physical and other clouds), analyses **dependencies**, produces **assessments** with readiness, rightsizing and cost, builds a **business case**, and executes **migrations** of servers and web apps, while integrating Microsoft and partner tools. It now offers application and cross-workload views and an **Azure Copilot migration agent** (Preview) for conversational planning. For architects and consultants, Azure Migrate data is the evidence base for strategy, wave plans and estimates.

## Key concepts

### Phases and capabilities

| Phase | Capability |
|---|---|
| **Decide** | Discovery via the **Azure Migrate appliance** (recommended, continuous) or **Azure Migrate Collector** (snapshot discovery for air-gapped/restricted networks), CSV import; **business case** (TCO, year-over-year cash flow, utilisation insights, CapEx to OpEx) |
| **Plan** | **Assessments** for Azure VMs, Azure SQL (SQL MI, SQL DB, SQL on VMs), App Service/AKS for web apps, Azure VMware Solution; readiness, rightsizing (performance-based or as-is), cost; **dependency analysis**; application and wave planning |
| **Execute** | Migrate VMware (agentless or agent-based), Hyper-V and physical/other-cloud servers; agentless ASP.NET web app migration to App Service; Data Box for bulk data; integrations with DMS and partner tools |

Azure Migrate is free (partner tools may charge); some discovery data (dependency visualisation) has free periods.

### Discovery

| Method | Notes |
|---|---|
| **Appliance** | Lightweight VM in your datacentre collecting configuration and performance data continuously; discovers software inventory, SQL instances, web apps, and agentless dependencies |
| **Collector** | Snapshot discovery without continuous Azure connectivity |
| **Import** | CSV/RVTools-style import for quick estimates (less accurate sizing) |

Collect performance data for at least a few weeks, including peak periods, for reliable rightsizing.

### Assessment settings

| Setting | Effect |
|---|---|
| **Sizing criteria** | Performance-based (recommended) or as-on-premises |
| **Percentile utilisation and comfort factor** | Headroom on rightsized targets |
| **Pricing options** | PAYG, reservations, savings plans, Azure Hybrid Benefit, EA/dev-test offers |
| **Target region and VM series** | Constrain recommendations |
| **Confidence rating** | Based on collected performance data completeness |

### Dependency analysis

**Agentless dependency analysis** is generally available for VMware, Hyper-V, physical and other-cloud servers, identifying network connections between servers to group them into applications and waves. Agent-based dependency analysis exists only in the classic view, which is scheduled for deprecation by the end of 2026, so use agentless analysis in the new experience.

### Azure Copilot migration agent (Preview)

A planning-focused conversational agent using your Azure Migrate project data: explore inventory, assess readiness, compare migration strategies, review costs and plan, with outputs you verify in the portal.

## How it works on Azure

Setup sequence:

1. Create an **Azure Migrate project** in a chosen geography (metadata location).
2. Deploy the **appliance** (OVA for VMware, VHD for Hyper-V, installer script for physical) and register it with the project; provide vCenter/Hyper-V and guest credentials for software inventory and dependencies.
3. Wait for discovery and collect performance data (2–4 weeks recommended).
4. Enable **agentless dependency analysis** for server groups.
5. Create **applications** and **groups**; run assessments for each target type.
6. Build a **business case**; review with finance.
7. Plan **waves**; replicate and migrate with the Migrate and Modernize tool (file 03).

Querying exported assessment data (example columns) into a planning workbook:

```text
Server | Readiness | Recommended size | Monthly cost (PAYG) | With AHB | With 3y RI | Confidence | Dependencies group
web01  | Ready     | D2s_v5           | €140                | €95      | €60        | 4 stars    | Orders
sql01  | Ready with conditions (SQL MI recommended) | GP 8 vCore | €1,150 | €820 | — | 4 stars | Orders
```

Use "Ready with conditions" and "Not ready" reasons (unsupported OS, boot type, disk size) to drive remediation tasks before migration.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Appliance vs collector vs import** | Appliance for accurate, continuous data; collector for restricted networks; import for very early estimates. |
| **Performance-based sizing** | Default; as-on-premises sizing for conservative lift-and-shift or missing data. |
| **Assessment targets** | Run multiple (VM vs PaaS vs AVS) for the same group to compare strategies. |
| **Business case inputs** | Validate on-premises cost assumptions with finance; they dominate TCO results. |
| **Project geography** | Choose for metadata residency; it doesn't restrict target regions. |
| **Classic view** | Move to the new experience; classic is being deprecated. |

## Hands-on

1. Create an Azure Migrate project and deploy an appliance in a lab (VMware or Hyper-V) or import a CSV inventory.
2. Enable agentless dependency analysis and group servers into an application.
3. Run Azure VM and Azure SQL assessments for the same group with AHB and 3-year reservation pricing.
4. Create a business case and review the TCO and cash-flow outputs.
5. Use the migration agent (if available) to ask which servers block readiness and why.

## Security, cost and reliability notes

- **Credentials**: appliance uses read-only accounts where possible; store securely; rotate after discovery.
- **Network**: appliance needs outbound access to Azure endpoints (or private endpoint connectivity).
- **Data**: metadata (configuration, performance, dependencies) is stored in the project geography.
- **Cost**: Azure Migrate is free; migration replication and target resources incur costs.

## Common mistakes

| Mistake | Fix |
|---|---|
| Sizing from a few days of data | Collect 2–4 weeks including peaks |
| Ignoring dependencies | Agentless dependency analysis |
| Only assessing for VMs | Compare PaaS and AVS targets |
| Trusting default on-premises costs in the business case | Validate inputs with finance |
| Staying on the classic view | New experience before deprecation |

## Check yourself

1. What does Azure Migrate do in the decide, plan and execute phases?
2. Compare appliance, collector and import discovery.
3. What assessment settings affect recommendations?
4. Which dependency analysis option should you use, and why?
5. What does the business case show?
6. What is the Azure Copilot migration agent?

<details>
<summary>Answers</summary>

1. Decide: discovery and business case; plan: assessments, dependency analysis, application/wave planning; execute: server and web app migrations and tool integrations.
2. Appliance: continuous, detailed discovery; collector: snapshot discovery for restricted networks; import: quick, less accurate.
3. Sizing criteria, percentile and comfort factor, pricing options (PAYG, reservations, savings plans, AHB), target region and VM series.
4. Agentless dependency analysis; it's GA across server types and agent-based is classic-only and being deprecated.
5. TCO on-premises vs Azure, year-over-year cash flows, utilisation-based insights and CapEx-to-OpEx effects.
6. A Preview conversational planning agent that uses project data to explore inventory, readiness, strategies and costs.

</details>

## References

- [What is Azure Migrate?](https://learn.microsoft.com/azure/migrate/migrate-services-overview)
- [Azure Migrate appliance](https://learn.microsoft.com/azure/migrate/migrate-appliance)
- [Dependency analysis](https://learn.microsoft.com/azure/migrate/concepts-dependency-visualization)
- [Assessment calculations](https://learn.microsoft.com/azure/migrate/concepts-assessment-calculation)
- [Business case calculation](https://learn.microsoft.com/azure/migrate/concepts-business-case-calculation)
