# Power BI and Semantic Models

## Why it matters

Power BI is how most organisations consume data, and the **semantic model** under a report is the governed business logic: measures, relationships, definitions of "revenue" and "active customer". That logic is now also **knowledge for AI**. Copilot in Power BI, Fabric data agents and Fabric IQ ask questions through semantic models. A well-modelled semantic model makes AI answers correct, and a messy one makes them confidently wrong. Consultants are asked about Power BI licensing, Direct Lake and embedding constantly.

## Key concepts

### Building blocks

| Item | Meaning |
|---|---|
| **Semantic model** (formerly dataset) | Tables, relationships, measures (DAX), hierarchies, security roles, and a storage mode |
| **Report** | Interactive pages of visuals built on one semantic model |
| **Dashboard** | A single page of pinned tiles from reports (legacy-style; reports and apps are more common) |
| **Paginated report** | Pixel-perfect, printable reports (RDL) |
| **App** | Packaged distribution of reports and dashboards to audiences |
| **Scorecard / metrics** | Goal tracking linked to data |

### Storage modes

| Mode | How it works | Freshness | Performance | Notes |
|---|---|---|---|---|
| **Import** | Data compressed into the in-memory VertiPaq engine on refresh | As of last refresh | Fastest | Model size limits per licence and capacity; refresh costs CUs |
| **DirectQuery** | Queries sent to the source at run time | Real time | Depends on source | Limits on DAX and modelling; source load |
| **Direct Lake** | Reads **Delta tables in OneLake** directly into memory on demand, without import refresh | Near real time ("framing" picks up new Delta versions) | Close to Import | Requires Fabric capacity and Delta tables in OneLake; can fall back to DirectQuery in some modes when limits are exceeded |
| **Composite** | Mix of modes per table | Mixed | Mixed | Powerful but complex |

**Direct Lake** comes in two flavours (checked 2026-11): **Direct Lake on SQL endpoints**, which uses the SQL analytics endpoint and can fall back to DirectQuery, and **Direct Lake on OneLake**, which reads Delta directly, doesn't fall back, can combine tables from several items, and supports OneLake security. Capacity SKU **guardrails** (maximum rows per table, model size in memory) decide whether a model fits.

### Modelling essentials

| Practice | Why |
|---|---|
| **Star schema**: facts (events, numbers) and dimensions (who, what, when, where) | Simplest, fastest and most predictable for DAX and AI |
| One-to-many relationships, single direction by default | Avoid ambiguity; use bidirectional filters sparingly |
| A proper **date table** marked as date table | Time intelligence functions depend on it |
| **Explicit measures** for every number users need | Consistent definitions; AI and Copilot use measures and their descriptions |
| Hide technical columns, name things in business language, add **descriptions** and **synonyms** | Improves discoverability for people and AI ("linguistic schema") |
| Remove unused columns; prefer integers for keys | Smaller, faster models |

### DAX basics

DAX (Data Analysis Expressions) defines calculated measures, columns and tables. The key concept is **filter context**: a measure is evaluated under the filters coming from slicers, rows and columns of the visual.

```dax
Total Revenue = SUM ( Sales[Amount] )

Revenue LY =
CALCULATE ( [Total Revenue], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )

Revenue YoY % =
DIVIDE ( [Total Revenue] - [Revenue LY], [Revenue LY] )

Revenue Enterprise =
CALCULATE ( [Total Revenue], Customer[Segment] = "Enterprise" )
```

| Function | Purpose |
|---|---|
| `CALCULATE` | Evaluate an expression in a modified filter context (the most important DAX function) |
| `FILTER`, `ALL`, `REMOVEFILTERS`, `KEEPFILTERS` | Control filter context |
| `SUMX`, `AVERAGEX` (iterators) | Row-by-row evaluation over a table |
| `DIVIDE` | Safe division (handles divide by zero) |
| `TOTALYTD`, `SAMEPERIODLASTYEAR`, `DATEADD` | Time intelligence |
| `RELATED` | Look up a value from the one-side of a relationship |
| Variables (`VAR ... RETURN`) | Readability and performance |

### Security in the model

- **Row-level security (RLS):** roles with DAX filters (for example `[Region] = USERPRINCIPALNAME()` via a mapping table).
- **Object-level security (OLS):** hide tables or columns from roles.
- **Sensitivity labels** (Microsoft Purview) flow from source to report to exported files.

### Power BI Embedded

| Scenario | Description | Capacity | Licences |
|---|---|---|---|
| **Embed for your organisation** ("user owns data") | Internal users sign in with Entra ID; content embedded in an internal app, SharePoint, Teams | F, P or A/EM SKU | Viewers need Pro/PPU unless content is on F64+ or P |
| **Embed for your customers** ("app owns data") | External users authenticated by your app; a service principal or master user calls Power BI | F, A/EM or P SKU | End users need no Power BI licence |

Use a **service principal** (or service principal profiles for multi-tenant ISVs) and generate **embed tokens**. Azure A SKUs only support Power BI items. F SKUs support Power BI embedding plus all of Fabric.

### Copilot in Power BI

Copilot can create report pages, summarise visuals, answer questions about data (through the semantic model), suggest DAX, and write measure descriptions. There is also a standalone Copilot experience for asking across the content you can access.

Requirements (checked 2026-11): workspaces on a paid **F2 or larger** Fabric capacity or **P1 or larger** Premium capacity (not trial), with Copilot enabled in tenant settings. Cross-geo processing settings may be needed where Azure OpenAI isn't available in the capacity's region. Copilot usage consumes capacity CUs.

Quality depends on the model. Microsoft recommends preparing models for AI: clear names, descriptions, synonyms, **AI instructions**, **verified answers** and an **AI data schema** limiting which fields Copilot uses (check the current "prep data for AI" features).

## How it works on Azure

### Direct Lake on a gold layer

```text
Lakehouse gold tables (Delta, V-Order)  ──►  Semantic model (Direct Lake)  ──►  Reports, Copilot, data agents
       ▲  nightly notebook writes new Delta version
       └── model "framing" picks up the new version automatically (or on refresh), no import needed
```

Keep gold tables **V-Ordered**, compacted and within SKU guardrails to avoid fallback or failures.

### Embed a report (Python, service principal)

```python
import requests
from azure.identity import ClientSecretCredential   # or a managed identity / certificate

cred = ClientSecretCredential("<tenant-id>", "<app-id>", "<secret-from-key-vault>")
token = cred.get_token("https://analysis.windows.net/powerbi/api/.default").token
h = {"Authorization": f"Bearer {token}"}

r = requests.post(
    "https://api.powerbi.com/v1.0/myorg/GenerateToken",
    headers=h,
    json={"reports": [{"id": "<report-id>"}],
          "datasets": [{"id": "<dataset-id>"}],
          "identities": [{"username": "customer-42", "roles": ["Tenant"], "datasets": ["<dataset-id>"]}]})
embed_token = r.json()["token"]   # pass to the front end with the embed URL (powerbi-client JS)
```

The service principal must be allowed by the tenant setting for service principals to use Power BI APIs and be a member of the workspace. Here a certificate or Key Vault-held secret is used. Prefer certificate or federated credentials over client secrets.

### Deployment and ALM

- Save models and reports as **Power BI Project (PBIP)** files with the **TMDL** model format for Git-friendly diffs.
- Use Fabric **Git integration** and **deployment pipelines** (dev → test → prod).
- Use the **XMLA endpoint** (Premium/Fabric) with Tabular Editor or SSMS for advanced management.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Import vs Direct Lake vs DirectQuery | Direct Lake for Fabric-native gold data at scale; Import for small or non-Fabric sources and maximum DAX flexibility; DirectQuery only for true real-time needs or very large sources you can't move |
| One big model vs many | Shared, certified "golden" models per domain reused by many reports, instead of a model per report |
| RLS in model vs in source | Model RLS for Power BI consumers. Note that Direct Lake on SQL endpoints and DirectQuery can use source security, so align with OneLake and Warehouse security |
| Embedding SKU | A SKUs for Power BI-only ISV embedding; F SKUs if you also need Fabric workloads or want one capacity |
| Copilot readiness | Invest in naming, descriptions and measures before enabling Copilot or data agents for business users |

## Hands-on

1. In Power BI Desktop, build a star schema model from sample data with a marked date table and the four DAX measures above.
2. Add RLS by region and test it with **View as role**.
3. In Fabric, create a **Direct Lake** semantic model on gold lakehouse tables and build a report on it.
4. Add descriptions and synonyms to tables and measures, enable Copilot (on an eligible capacity), and compare its answers before and after preparing the model.
5. Save the model as **PBIP/TMDL** and commit it to Git.

## Security, cost and reliability notes

- **Security:** apply RLS/OLS and sensitivity labels; restrict **Build** permission (which lets people create new content on a model) and export settings.
- **Security:** for "app owns data" embedding, never expose the service principal's credential to the browser. Only short-lived embed tokens go to the client.
- **Cost:** licensing dominates. Below F64, every viewer needs Pro or PPU, and embedding for external customers avoids per-user licences.
- **Cost:** large Import refreshes and Copilot usage consume CUs. Direct Lake avoids import refresh cost.
- **Reliability:** monitor refresh failures and Direct Lake fallback; use endorsement (promoted, certified) so people use trusted models.

## Common mistakes

- Flat, wide tables instead of a star schema, so DAX gets complicated and Copilot gets confused.
- Implicit measures (dragging numeric columns) instead of explicit measures.
- Bidirectional relationships everywhere.
- Ignoring Direct Lake guardrails, then getting unexpected DirectQuery fallback or errors.
- Assuming Copilot works on trial capacities or below F2.
- Embedding with a master user account instead of a service principal.

## Check yourself

1. Compare Import, DirectQuery and Direct Lake.
2. What does `CALCULATE` do?
3. What's the difference between "embed for your organisation" and "embed for your customers"?
4. What capacity is required for Copilot in Power BI?
5. Why does a star schema help AI features?
6. Which licence do viewers need on an F32 capacity?

<details>
<summary>Answers</summary>

1. Import loads data into memory on refresh (fast, but only as fresh as the last refresh). DirectQuery queries the source live (fresh, but source-dependent performance). Direct Lake reads Delta tables in OneLake directly into memory (near real time, import-like speed, Fabric only, subject to guardrails).
2. It evaluates an expression in a modified filter context, adding, removing or replacing filters.
3. "For your organisation": users authenticate with Entra ID and need appropriate licences. "For your customers": the app authenticates with a service principal and generates embed tokens, and end users need no Power BI licence.
4. A paid Fabric capacity F2 or larger, or Power BI Premium P1 or larger (not trial), with Copilot enabled by the tenant admin.
5. Clear facts, dimensions, relationships and named measures make the model's meaning explicit, so natural-language questions map reliably to the right fields and calculations.
6. Power BI Pro or Premium Per User (or an individual trial), because F32 is below the F64 threshold.

</details>

## References

- [What is Power BI?](https://learn.microsoft.com/power-bi/fundamentals/power-bi-overview)
- [Direct Lake overview](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Star schema guidance](https://learn.microsoft.com/power-bi/guidance/star-schema)
- [DAX overview](https://learn.microsoft.com/dax/dax-overview)
- [Row-level security](https://learn.microsoft.com/fabric/security/service-admin-row-level-security)
- [Power BI embedded analytics](https://learn.microsoft.com/power-bi/developer/embedded/embedded-analytics-power-bi)
- [Copilot in Power BI overview](https://learn.microsoft.com/power-bi/create-reports/copilot-introduction)
- [Prepare your data for AI](https://learn.microsoft.com/power-bi/create-reports/copilot-prepare-data-ai)
- [Power BI Desktop projects (PBIP)](https://learn.microsoft.com/power-bi/developer/projects/projects-overview)
