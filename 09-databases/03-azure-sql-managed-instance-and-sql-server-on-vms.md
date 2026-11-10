# Azure SQL Managed Instance and SQL Server on VMs

## Why it matters

Most enterprises have large SQL Server estates, and moving them to Azure is one of the most common migration projects consultants and architects run. Azure SQL Database suits new apps, but existing apps often rely on instance-level features it does not have. **Azure SQL Managed Instance** provides near-full SQL Server compatibility as PaaS; **SQL Server on Azure VMs** provides complete control as IaaS. Choosing between them, and using licensing benefits correctly, can change a business case by a large margin.

## Key concepts

### The Azure SQL family

| | **Azure SQL Database** | **Azure SQL Managed Instance** | **SQL Server on Azure VMs** |
|---|---|---|---|
| Model | PaaS, database-scoped | PaaS, instance-scoped | IaaS |
| Compatibility with SQL Server | High at database level | **Near 100%** at instance level | 100% |
| SQL Agent, Database Mail, linked servers, cross-database queries, CLR, Service Broker | Limited or no | **Yes** | Yes |
| OS access | No | No | Yes |
| Patching, backups, HA | Managed | Managed | You (with help from the SQL IaaS Agent extension) |
| Version | Always latest engine | Choose update policy | Any supported version you install |
| Networking | Public endpoint or private endpoint | Deployed into a dedicated subnet in your VNet | Your VNet |
| Typical use | New apps, modernised apps | Lift-and-shift with minimal change | Apps needing OS control, specific versions or unsupported features, third-party certifications |

### Azure SQL Managed Instance

| Topic | Detail |
|---|---|
| **Service tiers** | **General Purpose** (remote storage), **Next-gen General Purpose** (improved storage with independently configurable IOPS and higher limits; check current status), **Business Critical** (local SSD, Always On replicas, readable secondary) |
| Size | Up to tens of TB depending on tier; many databases per instance |
| **Instance pools** | Pre-provisioned capacity to host many small instances cost-effectively |
| **Update policy** | **SQL Server 2022** or **SQL Server 2025** update policy (engine aligned with that boxed version, enabling restore of backups back to on-premises SQL Server of that version) or **Always-up-to-date** (latest features) |
| **Managed Instance link** | Distributed availability group replication between SQL Server (2016 and later) and Managed Instance: near-real-time migration, read scale-out in Azure, and DR in either direction |
| Networking | Dedicated delegated subnet; public endpoint optional; private endpoints supported |
| Start and stop | Stop instances to save compute cost (General Purpose) |
| Business continuity | Zone redundancy, failover groups, point-in-time restore, long-term retention |
| Entra | Entra authentication, including Windows authentication for Entra principals via Kerberos |

Features still differ in places (for example some file system access, certain trace flags, FILESTREAM); check the compatibility list against your application.

### SQL Server on Azure VMs

| Topic | Detail |
|---|---|
| Images | Marketplace images with SQL Server pre-installed (pay-as-you-go licence) or bring-your-own-licence |
| **SQL IaaS Agent extension** | Registers the VM as a "SQL virtual machine" resource: licence management, automated patching and backups, storage configuration, Defender for SQL, portal management. Install it on every SQL VM |
| Storage | Premium SSD v2 or Ultra Disk for data and log; separate disks; host caching read-only for data; tempdb on local SSD |
| HA | Always On availability groups (across zones), failover cluster instances with shared disks or Premium file shares |
| VM sizing | Memory-optimised and constrained-vCPU sizes (to reduce per-core licensing) |
| Extended Security Updates | **Free** ESUs for SQL Server versions past end of support when running on Azure (for the ESU period) |

### Licensing

| Option | Detail |
|---|---|
| Pay-as-you-go | Licence included in the hourly price |
| **Azure Hybrid Benefit (AHB)** | Use existing SQL Server licences with Software Assurance (or subscriptions) to pay only for compute. Large savings on vCore-based Azure SQL and SQL VMs |
| **Free passive DR replica** | Licence-free passive secondary for HA and DR (on SQL VMs, and as a licence-free DR replica for Managed Instance failover groups when the secondary is used only for DR) |
| Reservations | Discount on compute for one or three years; combinable with AHB |
| Constrained vCPU VMs | Fewer licensed cores with the same memory and I/O |
| Developer and Express editions | Free for dev and test or small workloads on VMs |

AHB plus reservations commonly cut SQL compute costs by more than half compared with pay-as-you-go. Always ask a customer what licences and Software Assurance they hold.

## How it works on Azure

### Choosing the target

```text
Does the app need OS access, unsupported features, or a specific old version?
 ├─ Yes → SQL Server on Azure VMs (with SQL IaaS Agent extension)
 └─ No → Does it need instance features (SQL Agent, cross-db queries, CLR, linked servers, Service Broker)?
          ├─ Yes → Azure SQL Managed Instance
          └─ No → Azure SQL Database (consider Hyperscale for large databases)
```

Use **Azure Migrate** and its SQL assessment to evaluate an estate at scale: it discovers instances, checks compatibility with each target, recommends SKUs from performance data, and estimates cost (file 13).

### Managed Instance link migration

```text
On-premises SQL Server 2019 (primary)
   │ distributed availability group (near-real-time, over VPN/ExpressRoute)
   ▼
Azure SQL Managed Instance (readable secondary)
   → test and offload reads in Azure
   → cutover: planned failover makes MI the primary (minimal downtime)
   → optional: keep link for DR back to on-premises
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| MI vs SQL Database | MI when instance features or cross-database queries are needed and refactoring is not planned; SQL Database for modernised apps |
| MI vs VMs | MI to remove patching, backup and HA work; VMs for OS-level needs or unsupported features |
| MI tier | General Purpose (or next-gen) by default; Business Critical for latency-sensitive OLTP and a free readable replica |
| Update policy | SQL Server 2022 or 2025 policy if you need to restore back to on-premises; Always-up-to-date otherwise |
| Migration method | MI link for minimal downtime from SQL Server 2016+; backup and restore or DMS otherwise |
| Licensing | Apply AHB wherever licences exist; use the free DR replica |

## Hands-on

Managed Instance takes a while to deploy and has a minimum cost; these exercises focus on assessment and costing.

1. In the Azure pricing calculator, price a 16 vCore General Purpose Managed Instance with and without Azure Hybrid Benefit, then with a three-year reservation. Record the percentages saved.
2. Price an equivalent SQL Server Enterprise VM (memory-optimised, 16 vCPU) with pay-as-you-go licensing, then with AHB, then with a constrained-vCPU size licensed for 8 cores.
3. Read the Managed Instance T-SQL differences page and list five features your organisation's databases use, marking each as supported or not.
4. If you have an environment with SQL Server, install the Azure Migrate appliance or use the Azure SQL migration tooling to run a compatibility assessment.

```bash
# Inspect what SQL VM images are available
az vm image list --publisher MicrosoftSQLServer --all --query "[?contains(offer,'sql2022')].{offer:offer, sku:sku}" -o table | sort -u | head -20
```

## Security, cost and reliability notes

- **Security:** Entra authentication and managed identities where the app allows; Windows authentication for Entra principals on MI helps legacy apps.
- **Security:** MI lives in your VNet; restrict its subnet NSG and disable the public endpoint.
- **Security:** on VMs, you own OS hardening, patching and backup encryption. Use the SQL IaaS Agent extension and Defender for SQL.
- **Cost:** AHB, reservations, the free DR replica and stopping non-production MIs are the main levers.
- **Reliability:** zone-redundant MI or availability groups across zones for VMs; failover groups or distributed AGs for regional DR.

## Common mistakes

- Choosing SQL Database for an app that uses cross-database queries and SQL Agent, then rewriting under pressure.
- Choosing VMs by default and taking on patching and HA work that MI would handle.
- Forgetting Azure Hybrid Benefit, overpaying substantially.
- Paying for a licensed DR replica that qualifies as free.
- SQL VMs without the IaaS Agent extension, losing automated features and licence tracking.
- Undersized subnets for Managed Instance, blocking scaling.

## Check yourself

1. Name four features that make Managed Instance necessary instead of SQL Database.
2. What does the SQL IaaS Agent extension provide?
3. What is Azure Hybrid Benefit and what does it require?
4. What does the Managed Instance link do?
5. Why would you choose the SQL Server 2022 update policy for MI?
6. When must you use SQL Server on VMs?
7. What security update benefit does running old SQL Server versions in Azure provide?

<details>
<summary>Answers</summary>

1. Any four of: SQL Agent, cross-database queries, linked servers, CLR, Service Broker, Database Mail, instance-level configuration.
2. Licence model management, automated patching and backups, storage configuration, Defender for SQL integration and portal management of the SQL instance.
3. It lets you use existing SQL Server licences with Software Assurance or qualifying subscriptions in Azure and pay only the compute rate.
4. It replicates databases from SQL Server to MI with distributed availability groups, enabling minimal-downtime migration, read scale-out and hybrid DR.
5. To keep engine compatibility with SQL Server 2022 so backups can be restored back to on-premises SQL Server 2022.
6. When the application requires OS access, unsupported features, a specific old version or a vendor certification only on VMs.
7. Free Extended Security Updates for versions past end of support, for the ESU period.

</details>

## References

- [What is Azure SQL?](https://learn.microsoft.com/azure/azure-sql/azure-sql-iaas-vs-paas-what-is-overview)
- [Azure SQL Managed Instance overview](https://learn.microsoft.com/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview)
- [T-SQL differences between SQL Server and Managed Instance](https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server)
- [Managed Instance update policy](https://learn.microsoft.com/azure/azure-sql/managed-instance/update-policy)
- [Managed Instance link](https://learn.microsoft.com/azure/azure-sql/managed-instance/managed-instance-link-feature-overview)
- [SQL Server on Azure VMs overview](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview)
- [SQL IaaS Agent extension](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-iaas-agent-extension-automate-management)
- [Azure Hybrid Benefit for SQL Server](https://learn.microsoft.com/azure/azure-sql/azure-hybrid-benefit)
- [Extended Security Updates for SQL Server](https://learn.microsoft.com/sql/sql-server/end-of-support/sql-server-extended-security-updates)
