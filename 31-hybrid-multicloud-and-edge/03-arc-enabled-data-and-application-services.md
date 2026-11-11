# Arc-Enabled Data and Application Services

## Why it matters

SQL Server is one of the largest and longest-lived workloads in most enterprises, and much of it will stay on-premises or in other clouds for years because of latency, regulation or application constraints. Licensing, security patching, Entra authentication, backups and best-practice checks for thousands of instances are hard to manage consistently. **SQL Server enabled by Azure Arc** brings those estates under Azure management, and **SQL Managed Instance enabled by Azure Arc** runs an Azure-like managed SQL service on any Kubernetes. Application services on Arc have changed: **App Service, Functions and Logic Apps on Arc-enabled Kubernetes retired on 31 March 2026**, so architects must know which hybrid application options remain.

## Key concepts

### SQL Server enabled by Azure Arc

| Capability | Detail |
|---|---|
| **Inventory** | All instances and databases across sites in Azure Resource Graph |
| **Licensing** | Pay-as-you-go SQL Server licensing billed through Azure; licence type tracking; Azure Hybrid Benefit-style management; **Extended Security Updates** for out-of-support SQL versions |
| **Best practices assessment** | Configuration checks for performance and security |
| **Microsoft Entra authentication** | Modern identity, managed identity for outbound Azure access, no stored secrets |
| **Defender for SQL** | Vulnerability assessment and threat protection |
| **Backups and point-in-time restore** | Automated backups managed from Azure (feature availability varies) |
| **Migration assessment** | Readiness and sizing for Azure SQL targets (Topic 30 file 04) |
| **Monitoring** | Performance dashboards and Azure Monitor integration |

Enabled by the Azure extension for SQL Server, automatically installed when Arc-enabled servers with SQL Server are discovered (configurable).

### SQL Managed Instance enabled by Azure Arc

Azure Arc-enabled data services currently provide **SQL Managed Instance** on Kubernetes: deployed via an Arc **data controller** on a custom location, with frequent updates, elastic scale on available capacity, self-service provisioning, high availability (Business Critical tier with availability groups), backups and Azure-consistent management. It suits sovereign or disconnected scenarios needing a managed SQL experience on your infrastructure.

### Application services

| Option | Status / use |
|---|---|
| **App Service, Functions, Logic Apps on Arc-enabled Kubernetes** | **Retired 31 March 2026** (no new installs since 30 September 2025); migrate to alternatives |
| **Containers on Arc-enabled Kubernetes / AKS on Azure Local** | Run containerised apps with GitOps, Dapr, KEDA |
| **Azure IoT Operations** | Industrial data plane on Arc Kubernetes (Topic 32) |
| **Azure Container Apps** | Azure-hosted; for on-premises, use Kubernetes directly |
| **Logic Apps Standard (self-hosted)** | Check current hybrid deployment options |
| **API Management self-hosted gateway** | Run API gateways on-premises/edge, managed from Azure |
| **Azure Machine Learning on Arc** | Train/deploy models on Arc Kubernetes (Topic 21) |
| **AI extensions** | Agents and Tools with Foundry Local, Video Indexer on Arc (file 07) |

## How it works on Azure

Set licence type and enable ESU for an Arc-enabled SQL Server (CLI, `arcdata`/`connectedmachine` extensions; verify current parameters):

```bash
az sql server-arc extension feature-flag set --name LicenseType --value PAYG \
  --resource-group rg-arc-sql --machine-name sqlhost01
```

(Exact commands evolve; the portal's SQL Server – Azure Arc blade exposes licence type, ESU, Entra authentication, best practices assessment and backup settings.)

Resource Graph: SQL estate by version and licence type:

```kusto
resources
| where type =~ "microsoft.azurearcdata/sqlserverinstances"
| extend version = tostring(properties.version), edition = tostring(properties.edition), licence = tostring(properties.licenseType)
| summarize instances = count() by version, edition, licence
```

Deploy SQL MI enabled by Azure Arc (outline): Arc-enable a Kubernetes cluster → create a custom location → deploy the data controller → `az sql mi-arc create` with tier and resources → connect with Entra or SQL authentication → configure backups and monitoring.

Hybrid API pattern: API Management self-hosted gateway container in a factory cluster serving local APIs with policies managed centrally in Azure; telemetry flows to Azure when connected.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Arc for SQL Server** | Onboard all non-Azure SQL Server for inventory, licensing visibility and security even if migration is planned. |
| **PAYG vs licences** | PAYG for variable or short-lived instances; owned licences with Software Assurance for steady estates. |
| **ESU** | Use Arc ESU to buy time; plan upgrades or migration. |
| **SQL MI on Arc vs SQL Server** | MI on Arc for managed, Kubernetes-based database platforms at scale; SQL Server for existing VM-based estates. |
| **App platform on-premises** | After the App Service on Arc retirement, use containers on Kubernetes (AKS on Azure Local) with GitOps. |
| **Disconnected** | Check which features work disconnected (indirect mode for data services, Azure Local disconnected operations). |

## Hands-on

1. Arc-enable a server running SQL Server; confirm the SQL extension discovers instances.
2. Run a best practices assessment and remediate two findings.
3. Enable Entra authentication for the instance.
4. Build the SQL estate Resource Graph query into a workbook.
5. If migrating off App Service on Arc, map each app to a container deployment with GitOps.

## Security, cost and reliability notes

- **Defender for SQL** and vulnerability assessment on all Arc SQL instances.
- **Licensing compliance**: Arc licence-type tracking supports audits; keep it accurate.
- **PAYG costs** are billed hourly per core through Azure.
- **Retired services**: confirm no remaining dependencies on App Service on Arc.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building new apps on App Service on Arc | Retired; use containers |
| Unknown SQL estate | Arc inventory |
| Out-of-support SQL without ESU | Arc ESU or upgrade |
| SQL logins with passwords everywhere | Entra authentication |
| Treating Arc data services as identical to Azure SQL MI | Check feature differences |

## Check yourself

1. List six capabilities of SQL Server enabled by Azure Arc.
2. What is SQL Managed Instance enabled by Azure Arc, and what does it require?
3. What happened to App Service on Arc-enabled Kubernetes?
4. Which hybrid application options remain?
5. How are ESUs for SQL Server delivered?
6. When would you choose PAYG licensing through Arc?

<details>
<summary>Answers</summary>

1. Any six: inventory, PAYG licensing and licence tracking, ESU, best practices assessment, Entra authentication, Defender for SQL, backups/PITR, migration assessment, monitoring.
2. A managed SQL MI running on Arc-enabled Kubernetes via a data controller on a custom location.
3. It retired on 31 March 2026 (no new installs from 30 September 2025).
4. Containers on Arc Kubernetes/AKS on Azure Local with GitOps, APIM self-hosted gateway, Azure IoT Operations, Azure ML on Arc, AI extensions.
5. Through Azure Arc, enabled and billed in Azure.
6. For variable, short-lived or newly provisioned instances where buying licences isn't efficient.

</details>

## References

- [SQL Server enabled by Azure Arc](https://learn.microsoft.com/sql/sql-server/azure-arc/overview)
- [Azure Arc-enabled data services](https://learn.microsoft.com/azure/azure-arc/data/overview)
- [App Service, Functions and Logic Apps on Azure Arc (retired)](https://learn.microsoft.com/azure/app-service/overview-arc-integration)
- [API Management self-hosted gateway](https://learn.microsoft.com/azure/api-management/self-hosted-gateway-overview)
