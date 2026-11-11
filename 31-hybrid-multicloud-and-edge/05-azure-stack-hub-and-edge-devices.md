# Azure Stack Hub and Edge Devices

## Why it matters

Some environments are not just "on-premises" but genuinely isolated or harsh: ships, military deployments, mines, oil platforms, disaster zones, factories with no reliable WAN. They need Azure-consistent services that run fully disconnected, or rugged appliances that process data locally and ship it to Azure when they can. The Azure portfolio for these scenarios includes **Azure Stack Hub** (an integrated system running a subset of Azure IaaS and PaaS services on-premises, connected or disconnected), **Azure Stack Edge** devices (hardware-as-a-service appliances for edge compute and data transfer) and **Azure Data Box** (offline data transfer). Architects need to know where each fits, how they relate to Azure Local, and their lifecycle status.

## Key concepts

### Portfolio

| Offering | What it is | Fit |
|---|---|---|
| **Azure Local** | Hyperconverged infrastructure managed via Arc; connected or disconnected operations (file 04) | Most new on-premises and edge deployments |
| **Azure Stack Hub** | Integrated system from OEM partners delivering Azure services (VMs, storage, networking, some PaaS) with its own Azure-consistent portal and ARM endpoint; can run fully disconnected | Existing deployments; disconnected cloud with Azure-consistent APIs where already invested |
| **Azure Stack Edge** (Pro 2, Pro GPU, Mini R) | Microsoft-supplied appliances (hardware as a service) with compute (Kubernetes/VMs), GPU options and data transfer to Azure Storage; ruggedised options | Edge inference, data pre-processing, field deployments |
| **Azure Data Box** family | Offline devices to move large datasets to/from Azure | Bulk migration and periodic transfers |
| **Azure Stack Edge Pro FPGA** | **End of life February 2024** | Migrate to Pro 2 or Pro GPU |

Lifecycle notes: App Service on Azure Stack Hub is retiring on 30 September 2029 (per the retirement notice); check each component's lifecycle on Microsoft Learn before designing. Microsoft's strategic direction for new on-premises infrastructure is Azure Local with Arc.

### Azure Stack Hub concepts

| Concept | Detail |
|---|---|
| **Operators vs users** | Operators run the stamp (capacity, updates, offers/plans/quotas, marketplace syndication); users consume subscriptions like Azure |
| **Identity** | Microsoft Entra ID (connected) or AD FS (disconnected) |
| **Services** | IaaS (VMs, managed disks, VNets, load balancer, VPN gateway), Storage (blobs, tables, queues), Key Vault, AKS engine-based options and selected PaaS resource providers (Event Hubs, SQL/MySQL adapters) depending on version |
| **API profiles** | Azure Stack Hub uses specific API versions; use API profiles in SDKs and templates |
| **Updates** | Microsoft and OEM updates applied by operators; must stay within supported update window |
| **Billing** | Pay-as-you-use (connected) or capacity model (disconnected) |

### Azure Stack Edge concepts

| Concept | Detail |
|---|---|
| **Ordering** | Via Azure portal (resource), device shipped by Microsoft |
| **Compute** | Kubernetes (Arc-enabled) and VMs on the device; GPUs (Pro GPU, Pro 2 with GPU) for inference |
| **Data transfer** | SMB/NFS shares that tier/upload to Azure Storage; local shares for edge processing |
| **Management** | Azure portal, local web UI; Arc for Kubernetes workloads |
| **Rugged** | Mini R for portable, battery-powered field use |

## How it works on Azure

Choosing an edge option:

```text
Need Azure-consistent infrastructure in your own datacentre or site?
  ├─ New deployment → Azure Local (connected; disconnected operations for sovereign/isolated)
  └─ Existing Azure Stack Hub with disconnected PaaS needs → continue, plan roadmap
Need a managed appliance for edge inference or data transfer at many small sites/field?
  └─ Azure Stack Edge (Pro 2 / Pro GPU / Mini R)
Need to move tens or hundreds of TB once or periodically without network?
  └─ Azure Data Box
```

Hybrid application pattern with Azure Stack Hub: deploy the same ARM templates (with the right API profile) to Azure and Azure Stack Hub; use Azure for global services and Hub for disconnected operations; sync data when connected via Storage replication or messaging.

Edge inference pattern with Azure Stack Edge: cameras → local Kubernetes inference container on GPU → events to local storage/IoT → upload summaries to Azure when connected → retrain models in Azure ML → redeploy via GitOps.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Strategic platform** | Azure Local for new; Azure Stack Hub where existing investment or specific disconnected PaaS requirements exist. |
| **Disconnected identity** | AD FS or local identity for disconnected Hub; plan certificate and identity operations carefully. |
| **Appliance vs owned hardware** | Azure Stack Edge (Microsoft-supplied, subscription) for many small sites; Azure Local on partner hardware for larger sites. |
| **Data gravity** | Process at the edge; send aggregated results to Azure to save bandwidth. |
| **Lifecycle risk** | Track retirement notices; avoid building new solutions on retiring components. |

## Hands-on

1. Compare Azure Local, Azure Stack Hub and Azure Stack Edge for three scenarios: factory, ship, retail chain.
2. Review the Azure Stack Hub user documentation and identify the API profile needed for a Bicep/ARM deployment.
3. Design an Azure Stack Edge Pro GPU inference pipeline with GitOps updates.
4. Plan a Data Box transfer of 200 TB, including encryption and chain of custody.
5. Check the lifecycle status of every edge component in your design.

## Security, cost and reliability notes

- **Physical security and tamper protection** for field devices; encrypt data at rest; secure shipping.
- **Patching discipline**: Azure Stack Hub must be updated within support windows.
- **Cost**: Hub hardware and capacity/usage billing; Stack Edge monthly device fees; Data Box per order.
- **Resilience**: single devices are single points of failure; plan local redundancy or graceful degradation.

## Common mistakes

| Mistake | Fix |
|---|---|
| New designs on retired devices (Pro FPGA) | Use current models |
| Assuming Azure Stack Hub equals Azure feature parity | Check supported services and API profiles |
| Ignoring update windows | Operator update discipline |
| Shipping raw data to Azure from every site | Process locally, send summaries |
| No plan for device failure | Spares and redundancy |

## Check yourself

1. Compare Azure Local, Azure Stack Hub and Azure Stack Edge.
2. What are operators and users in Azure Stack Hub?
3. What identity options does Azure Stack Hub support?
4. Which Azure Stack Edge model reached end of life, and what replaces it?
5. When would you use Azure Data Box?
6. Why are API profiles important for Azure Stack Hub?

<details>
<summary>Answers</summary>

1. Azure Local: Arc-managed HCI on partner hardware (strategic, connected/disconnected). Azure Stack Hub: integrated system with Azure-consistent portal/APIs and some PaaS, can be disconnected. Azure Stack Edge: Microsoft-supplied appliances for edge compute, inference and data transfer.
2. Operators manage the stamp, capacity, offers and updates; users consume subscriptions and deploy resources.
3. Microsoft Entra ID when connected, AD FS when disconnected.
4. Azure Stack Edge Pro FPGA (end of life February 2024); Pro 2 or Pro GPU.
5. For offline transfer of large datasets to or from Azure.
6. Azure Stack Hub supports specific API versions; profiles ensure templates and SDKs use compatible versions.

</details>

## References

- [Azure Stack Hub overview (operator)](https://learn.microsoft.com/azure-stack/operator/azure-stack-overview)
- [Azure Stack Hub user documentation](https://learn.microsoft.com/azure-stack/user/user-overview)
- [Azure Stack Edge Pro 2](https://learn.microsoft.com/azure/databox-online/azure-stack-edge-pro-2-overview)
- [Azure Stack Edge FPGA to GPU migration](https://learn.microsoft.com/azure/databox-online/azure-stack-edge-migrate-fpga-gpu)
- [Azure Data Box](https://learn.microsoft.com/azure/databox/data-box-overview)
