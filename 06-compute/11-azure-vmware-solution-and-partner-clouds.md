# Azure VMware Solution and Partner Clouds

## Why it matters

Not everything moves to Azure by being rebuilt on Azure services. Large enterprises run VMware estates with thousands of VMs, Oracle databases on Exadata, SAP landscapes and Nutanix clusters, and they often cannot or will not re-platform them on the timescale the business wants. Azure hosts these stacks natively so they can move as they are, sit next to Azure services, and be modernised later. Engineers rarely build these; consultants and architects are regularly asked about them, usually in a migration or data centre exit conversation, and increasingly because the data an AI project needs lives there.

Licensing and commercial terms in this area change often. Verify before quoting.

## Key concepts

### The pattern

Each offering in this file follows the same idea: a **third-party platform running on dedicated infrastructure inside Azure data centres**, sold and supported through Azure, connected to your virtual networks.

| What you gain | What you give up |
|---|---|
| Move workloads without re-architecting, re-training staff or re-licensing applications | Cloud-native elasticity and pricing: you buy hosts or racks, not seconds |
| Keep existing tools, skills and operational processes | Full use of Azure-native management for the workloads inside |
| Low-latency adjacency to Azure services (AI, analytics, backup) | Cost efficiency compared with a fully modernised design |
| A faster data centre exit | Some of the benefits of PaaS (patching, scaling) |

They are best understood as a **first step** or a **landing place for what cannot change**, not a destination for everything.

## Azure VMware Solution

### What it is

A **VMware private cloud** (vSphere, vSAN, NSX, vCenter, and HCX for migration) on dedicated bare-metal hosts in Azure, operated as a first-party Azure service. Microsoft manages the hosts, the VMware software lifecycle and the underlying infrastructure. You manage the VMs and their guest operating systems, using the VMware tools you already know.

| Aspect | Detail |
|---|---|
| **Unit of purchase** | Hosts, in clusters. Minimum **3 hosts** per cluster; several host SKUs with different core, memory and storage capacity |
| **Private cloud** | One or more clusters with a vCenter and NSX manager |
| **Storage** | vSAN on the hosts' local disks; extend with Azure NetApp Files datastores, Elastic SAN, or other supported external storage |
| **Networking** | Classic deployments connect to your VNets through an Azure-managed ExpressRoute circuit, and to on-premises with ExpressRoute Global Reach. Newer-generation private clouds deploy **directly into an Azure virtual network**, simplifying connectivity |
| **Migration** | **VMware HCX**: bulk migration, live vMotion, layer 2 network extension so VMs can keep their IP addresses during migration |
| **Operations** | vCenter and NSX for VM operations; Azure portal for the private cloud; Azure Arc integration to project VMs into Azure for governance |
| **Availability** | vSphere HA within a cluster; **stretched clusters** across availability zones in supported regions; a second private cloud in another region for disaster recovery (VMware Live Recovery, or partner tools) |
| **Pricing** | Per host per hour, with reserved instances for 1 or 3 years. Azure Hybrid Benefit for Windows Server and SQL Server; free extended security updates for eligible older Windows and SQL versions |

### VMware licensing after the Broadcom acquisition

Broadcom's changes to VMware licensing (subscription-only, bundled as VMware Cloud Foundation) reshaped the economics of VMware everywhere. Azure VMware Solution has adjusted accordingly, including requirements for customers to bring their own VMware Cloud Foundation subscription for new deployments in many cases, with VCF licence portability. The details have changed more than once. Confirm the current licensing model, what is included in the AVS price, and any deadlines, with Microsoft and Broadcom before building a business case.

### When it fits

| Good fit | Poor fit |
|---|---|
| A data centre exit on a deadline | A handful of VMs: three large hosts is the entry price |
| Hundreds or thousands of VMs that cannot be assessed and re-platformed individually in time | Workloads that will be modernised to PaaS anyway |
| Applications tied to VMware features, tooling or appliance formats | New applications |
| Disaster recovery for an on-premises VMware estate | Cost-sensitive, elastic workloads |
| Teams with deep VMware skills and little Azure experience | Organisations wanting to leave VMware licensing behind |
| Hardware refresh avoidance | |

The usual strategy is **migrate, then modernise**: move quickly with HCX, exit the data centre, then move suitable workloads to native Azure VMs and PaaS over time, shrinking the private cloud.

### AVS versus migrating to native Azure VMs

| | Azure VMware Solution | Native Azure VMs |
|---|---|---|
| Migration effort | Lowest: same hypervisor, tools, IPs | Per-VM assessment, conversion, re-addressing, re-tooling |
| Speed | Fast | Slower |
| Run cost | Host-based; efficient only when hosts are well packed; plus VMware licensing | Per VM, with autoscale, reservations, right-sizing |
| Operations | VMware skills and tools | Azure-native skills and tools |
| Modernisation | Deferred | Started |
| Flexibility | Constrained by host capacity | Elastic |

**Azure Migrate** assesses a VMware estate for both targets and produces a cost comparison.

## Nutanix Cloud Clusters on Azure

**NC2 on Azure** runs the Nutanix stack (AOS, AHV, Prism) on Azure bare-metal hosts, managed with Nutanix's own tooling and licensed through Nutanix. It serves Nutanix customers in the same way AVS serves VMware customers: extension, migration and disaster recovery without re-platforming. It is also considered by organisations moving off VMware to Nutanix. Regional availability and supported host types are more limited than AVS.

## Oracle Database@Azure

### What it is

Oracle database services running on **Oracle Cloud Infrastructure hardware physically located inside Azure data centres**, operated by Oracle and purchased through Azure.

| Aspect | Detail |
|---|---|
| **Services** | Exadata Database Service (dedicated and Exascale infrastructure), Autonomous Database, Base Database Service, and related Oracle services |
| **Placement** | In an Azure region and availability zone, attached to a **delegated subnet** in your virtual network, so applications on Azure reach the database with very low latency |
| **Purchase** | Through Azure Marketplace; counts toward Azure consumption commitments. Bring-your-own-licence and Oracle support rewards apply |
| **Management** | Provisioning and monitoring in the Azure portal; database administration through Oracle's tools and OCI console |
| **Integration** | Microsoft Entra ID, Azure Monitor, Key Vault, Defender, Purview, Fabric mirroring and other Azure services |
| **Support** | Joint Microsoft and Oracle support |

### Why it matters

Before this, Oracle Exadata workloads and features such as Real Application Clusters had no first-class home on Azure: the choices were self-managed Oracle on Azure VMs (no RAC), or keeping the database elsewhere and paying a latency penalty. Oracle Database@Azure lets the application tier modernise on Azure while the database stays on the platform it requires.

**AI relevance:** enterprise systems of record often sit in Oracle. With the database adjacent to Azure, an AI application can query it with low latency, data can be mirrored into Fabric or indexed for retrieval, and newer Oracle database releases include native vector search that some teams use alongside or instead of a separate vector store.

**Other options for Oracle on Azure:** Oracle Database on Azure VMs (self-managed, with Data Guard for resilience); the older Oracle Interconnect between Azure and OCI regions; or migrating to PostgreSQL where the application permits, which removes Oracle licensing altogether.

## SAP on Azure

SAP is among the largest enterprise workloads on Azure.

| Element | Notes |
|---|---|
| **Certified infrastructure** | SAP HANA-certified VM families, notably **M-series** with multiple terabytes of memory, and E-series for smaller systems; certified storage configurations using Premium SSD v2, Ultra Disk or Azure NetApp Files |
| **Azure Center for SAP solutions** | Azure service for deploying, registering and managing SAP systems as a logical unit, with quality checks and cost views |
| **SAP Deployment Automation Framework** | Open-source Terraform and Ansible automation for SAP landscapes |
| **RISE with SAP** | SAP's managed offering in which SAP operates S/4HANA on a hyperscaler, including Azure. The customer consumes it as a service and integrates with their own Azure landing zone |
| **Azure Monitor for SAP solutions** | Telemetry from HANA, NetWeaver, operating systems and clusters |
| **High availability and disaster recovery** | Pacemaker or Windows clustering, HANA System Replication, zone-redundant deployments, cross-region replication |
| **Integration** | Entra ID single sign-on, Sentinel for SAP, Fabric and Data Factory connectors, API Management and Logic Apps for SAP interfaces |

SAP on Azure is a specialism with its own certification (AZ-120). For AI projects, what matters is **integration**: SAP holds the business data and processes that agents and copilots need to read and act on, through OData and BAPI interfaces, connectors, or SAP's own AI layer. Topic 34 returns to enterprise system integration.

## Other specialised platforms

| Offering | What it is |
|---|---|
| **Azure Large Instances / BareMetal Infrastructure** | Dedicated, certified bare-metal for specialised workloads, historically SAP HANA at the largest sizes and Epic healthcare systems |
| **Mainframe and midrange modernisation** | Partner solutions and patterns for rehosting or refactoring IBM Z and IBM i workloads; AI-assisted code analysis and conversion is an active area |
| **Azure Red Hat OpenShift** | Jointly managed OpenShift (topic 07) |
| **Azure NetApp Files** | NetApp ONTAP as a first-party storage service (topic 08) |
| **Azure Native ISV services** | Partner SaaS (Datadog, Elastic, Confluent, MongoDB Atlas, Dynatrace, Palo Alto and others) provisioned and billed through Azure |
| **Epic on Azure, Citrix and Omnissa Horizon on Azure** | Vendor-specific reference architectures and services |

## How it works on Azure

### Where these sit in a landing zone

They are **workload landing zones** with unusual contents (topic 03). Microsoft publishes landing zone accelerators for AVS, SAP, Oracle Database@Azure and others. Common integration points:

| Concern | Integration |
|---|---|
| **Network** | Connected to the hub; traffic inspected at the hub firewall where required; DNS integrated with the central resolver |
| **Identity** | Entra ID for the Azure management plane; the platform's own directory integration for workloads inside |
| **Governance** | Azure Policy and RBAC govern the Azure resource (the private cloud, the Exadata infrastructure). Inside the platform, use its own controls, extended with Azure Arc where available |
| **Monitoring** | Platform diagnostics to the central Log Analytics workspace |
| **Security** | Defender for Cloud and Sentinel connectors for the platform |
| **Backup and DR** | Platform-native or partner tools, with Azure storage as a target |

### Connecting these estates to AI workloads

The business reason to care in an AI context: **this is where the data is**.

| Pattern | Description |
|---|---|
| **Query in place** | An agent tool or API calls the system directly (SQL, OData, REST) over private networking. Simple and fresh; adds latency and load, and needs careful authorisation |
| **Replicate to an analytics store** | Mirror or pipeline data into Fabric, a lakehouse or a database, then index or query that copy. Decouples AI workloads from the transactional system |
| **Index for retrieval** | Extract documents and records into Azure AI Search with access control metadata |
| **Expose as governed APIs** | API Management in front of the system, with authentication, quotas and logging. The safest way to let agents act on a system of record |
| **Vendor AI features** | Use the platform's own assistants and vector capabilities where they suffice |

Whichever is chosen: the transactional system's performance must be protected, user-level permissions must be carried through, and write actions by agents need approval controls.

### Building a business case

For AVS, NC2 or Oracle Database@Azure against alternatives, compare over three to five years:

1. **Current cost**: hardware refresh due, data centre, licences and support, staff, power.
2. **Target run cost**: hosts or infrastructure (reserved), licences (bring your own, portability, hybrid benefit), networking, storage, backup, Azure services.
3. **Migration cost and duration**, including dual running.
4. **Risk and time value** of the deadline (lease expiry, contract renewal, end of support).
5. **Modernisation path**: what moves to native services afterwards, and when the host footprint shrinks.

Use Azure Migrate's business case output as a starting point, then correct it with real licence terms.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| AVS or native Azure VMs | AVS for speed and scale of migration with a deadline. Native for small or modernising estates and long-term efficiency |
| How long to stay on AVS | Treat it as transitional for most workloads. Set modernisation milestones, or the host bill becomes permanent |
| Oracle: Database@Azure, VMs or migrate | Database@Azure for Exadata and RAC dependencies. VMs for simpler databases. PostgreSQL migration where feasible to remove licence cost |
| SAP: self-managed on Azure or RISE | Depends on the customer's SAP contract and how much they want to operate. Either way, design the surrounding landing zone and integration |
| Host sizing | Size for memory and storage with realistic consolidation ratios. An extra host is a large step cost |
| Disaster recovery | A second region for these platforms roughly doubles the cost. Tier workloads; not everything needs it |

## Hands-on

Exploration and estimation; these platforms need capacity requests and are expensive.

1. In the Azure portal, open the create experience for **Azure VMware Solution**. Note the host SKUs, minimum cluster size, the address block requirement (a `/22` for management) and the prerequisites. Cancel.
2. In the **pricing calculator**, estimate a three-host AVS private cloud for a month, pay-as-you-go and with a three-year reservation. Compare with 60 general-purpose VMs of 4 vCPUs and 16 GiB on native Azure with a savings plan.
3. Search the portal for **Oracle Database@Azure** and read the onboarding steps and supported regions.
4. Open **Azure Migrate → Discover, assess and migrate** and review the assessment types: Azure VM, Azure VMware Solution, Azure SQL, and the business case feature.
5. Find the landing zone accelerator documentation for AVS and for SAP in the Cloud Adoption Framework. For each, list three design areas that differ from a standard workload landing zone.

Consulting exercise: a manufacturer has 900 VMware VMs, a data centre lease ending in ten months, two Oracle RAC databases behind its ERP, and a new mandate to "do something with AI on our operational data". Write a one-page recommendation covering the migration target for the VMs, the Oracle databases, how the AI initiative gets data, what happens in the twelve months after migration, and the three biggest risks.

<details>
<summary>Notes</summary>

A defensible answer: migrate the VMware estate to Azure VMware Solution with HCX to meet the lease deadline, after confirming current VCF licensing terms; move the Oracle RAC databases to Oracle Database@Azure in the same region; land both in a landing zone with ExpressRoute during migration. For AI, replicate ERP data into Fabric or a lakehouse and expose key transactions through API Management, so the AI workload does not load the ERP directly. In the following year, assess and move suitable VMs to native services to shrink the host count. Risks: licensing changes altering the business case, migration timeline against the lease date, and data quality and access control for the AI use cases.

</details>

## Security, cost and reliability notes

- **Security:** governance inside these platforms is not automatically Azure governance. Azure Policy does not reach VMs inside a VMware private cloud unless you extend it with Arc. Plan patching, endpoint protection and monitoring for the guests explicitly.
- **Security:** apply the same network isolation and identity standards as any workload. These estates often carry the most sensitive legacy systems and the oldest operating systems.
- **Cost:** you pay for hosts and infrastructure whether or not they are full. Utilisation is the main cost lever; reservations the second.
- **Cost:** licensing terms dominate the business case and are the least stable input. Validate them.
- **Reliability:** understand what the provider manages (hosts, platform software) and what remains yours (guest OS, application, data protection, disaster recovery).
- **Reliability:** capacity for these services is regional and finite. Engage early for large deployments.

## Common mistakes

- Treating AVS as the permanent home for everything, with no modernisation plan.
- Choosing AVS for a small estate that fits comfortably on native VMs.
- Business cases built on outdated VMware or Oracle licensing assumptions.
- Assuming Azure Policy, Defender and Backup cover the guests automatically.
- Under-sizing network address space and connectivity for HCX and management.
- Leaving Oracle elsewhere and paying a latency penalty on every application call.
- Letting AI agents query or write to a system of record directly, with no API layer, quotas or approvals.
- Forgetting disaster recovery cost until late in the design.
- Not using free extended security updates and hybrid benefits that the migration makes available.

## Check yourself

1. What is the common pattern behind Azure VMware Solution, NC2 and Oracle Database@Azure?
2. What does Microsoft manage in AVS, and what do you manage?
3. What is the minimum cluster size for AVS, and why does it matter?
4. What does VMware HCX provide during migration?
5. When is AVS a better choice than migrating to native Azure VMs, and when is it not?
6. What problem does Oracle Database@Azure solve?
7. Why do these platforms matter for AI projects?
8. What should a consultant verify before presenting a business case for any of them?

<details>
<summary>Answers</summary>

1. A third-party platform running on dedicated infrastructure inside Azure data centres, purchased and supported through Azure and connected to your virtual networks, so workloads can move without re-platforming.
2. Microsoft manages the physical hosts, the VMware software stack and its lifecycle. You manage the VMs, guest operating systems, applications and data.
3. Three hosts. It sets a high entry cost, making AVS uneconomic for small estates.
4. Bulk and live migration of VMs, and layer 2 network extension so workloads can keep their IP addresses during the move.
5. Better for large estates on a deadline, VMware-dependent applications and teams with VMware skills. Not for small estates, new applications, or workloads that will be modernised anyway.
6. It gives Exadata and RAC-based Oracle workloads a first-class, low-latency home next to Azure applications and services, operated by Oracle and bought through Azure.
7. Systems of record and enterprise data live in them, so AI applications need to query, replicate, index or act on that data, with adjacency reducing latency.
8. Current licensing terms and what is included in the price, regional availability and capacity, host or infrastructure sizing, and the modernisation path afterwards.

</details>

## References

- [What is Azure VMware Solution?](https://learn.microsoft.com/azure/azure-vmware/introduction)
- [Azure VMware Solution private cloud and cluster concepts](https://learn.microsoft.com/azure/azure-vmware/architecture-private-clouds)
- [VMware Cloud Foundation licence portability and AVS licensing](https://learn.microsoft.com/azure/azure-vmware/vmware-cloud-foundations-license-portability)
- [Azure VMware Solution landing zone accelerator](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/azure-vmware/)
- [About Nutanix Cloud Clusters on Azure](https://learn.microsoft.com/azure/baremetal-infrastructure/workloads/nc2-on-azure/about-nc2-on-azure)
- [Overview of Oracle Database@Azure](https://learn.microsoft.com/azure/oracle/oracle-db/database-overview)
- [Oracle on Azure landing zone accelerator](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/oracle-iaas/)
- [Use Azure to host and run SAP workload scenarios](https://learn.microsoft.com/azure/sap/workloads/get-started)
- [What is Azure Center for SAP solutions?](https://learn.microsoft.com/azure/sap/center-sap-solutions/overview)
- [SAP on Azure landing zone accelerator](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/sap/)
- [About Azure Migrate](https://learn.microsoft.com/azure/migrate/migrate-services-overview)
