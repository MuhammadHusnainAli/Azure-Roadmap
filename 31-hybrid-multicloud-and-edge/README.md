# 31 · Hybrid, Multicloud and Edge

> Tracks: E – A ● C ○ · Part E · Secure, Operate, Migrate and Extend

Azure doesn't stop at Azure regions. This topic covers Azure Arc as the control plane for servers, Kubernetes, SQL Server and data services anywhere; operating hybrid estates with policy, Defender, monitoring, Update Manager and GitOps; Azure Local (formerly Azure Stack HCI) including disconnected operations for sovereign private clouds; Azure Stack Hub, Azure Stack Edge and Data Box for isolated and rugged scenarios; multicloud management across AWS and GCP with Defender for Cloud, the multicloud connector, Sentinel and Entra; AI at the edge with Foundry Local and Agents and Tools with Foundry Local (local agentic RAG, Preview); and an awareness view of Azure Operator Nexus for telecom. It also flags retirements such as App Service on Arc-enabled Kubernetes (retired 31 March 2026).

## What you will be able to do

- Explain how Arc projects external resources into Azure and what it enables.
- Operate hybrid servers and Kubernetes consistently with Azure tools.
- Manage SQL Server estates with Arc, including licensing and ESU.
- Decide when Azure Local, Azure Stack Hub or Azure Stack Edge fits.
- Govern and secure AWS and GCP alongside Azure.
- Design AI that runs on devices and sites with cloud-based governance.
- Recognise telecom platform options at awareness level.

## Prerequisites

[Topic 03](../03-governance-and-landing-zones/README.md) (policy), [Topic 07](../07-containers-and-kubernetes/README.md) (Kubernetes), [Topic 25](../25-security/README.md) and [Topic 26](../26-monitoring-and-observability/README.md).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Azure Arc Overview](01-azure-arc-overview.md) | 30 min | Onboard a server and cluster |
| 02 | [Arc-Enabled Servers and Kubernetes](02-arc-enabled-servers-and-kubernetes.md) | 45 min | Policy, Update Manager, GitOps |
| 03 | [Arc-Enabled Data and Application Services](03-arc-enabled-data-and-application-services.md) | 35 min | SQL Server enabled by Arc |
| 04 | [Azure Local](04-azure-local.md) | 40 min | Size a cluster, Arc VM, AKS |
| 05 | [Azure Stack Hub and Edge Devices](05-azure-stack-hub-and-edge-devices.md) | 30 min | Choose edge options |
| 06 | [Multicloud Management](06-multicloud-management.md) | 40 min | AWS in Defender and Arc |
| 07 | [AI at the Edge](07-ai-at-the-edge.md) | 40 min | Foundry Local hybrid app |
| 08 | [Azure Operator Nexus and Telecom](08-azure-operator-nexus-and-telecom.md) | 20 min | Map telecom needs |

**Architects:** all of it. **Consultants:** 01, 04, 06 and 07 for positioning.

## Completion checklist

- [ ] I have onboarded servers and a Kubernetes cluster to Arc and governed them with policy.
- [ ] I patch and monitor hybrid servers with Update Manager and AMA.
- [ ] I can manage SQL Server licensing and security through Arc.
- [ ] I can size and justify an Azure Local deployment, including disconnected needs.
- [ ] I can choose between Azure Local, Stack Hub, Stack Edge and Data Box.
- [ ] I have connected AWS or GCP to Defender for Cloud and the multicloud connector.
- [ ] I have built an app that runs AI locally with Foundry Local.
- [ ] I can explain what Operator Nexus is for.

## Maps to

- **AZ-305**: design hybrid and multicloud solutions.
- **AZ-104** (adjacent): Arc-enabled server management.
- **SC-100** (adjacent): multicloud security strategy.

## Next

32 · [IoT, Industry and Specialised Services](../32-iot-industry-and-specialised-services/README.md)
