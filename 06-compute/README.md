# 06 · Compute

> Tracks: E ○ A ● C ○ · Part B · Core Infrastructure and Application Platform

Compute is where code runs. This topic covers infrastructure compute: virtual machines and everything built directly on them, including the GPU capacity behind self-hosted AI. Containers and Kubernetes are topic 07; App Service, Functions and other managed application hosts are topic 10. Most AI applications should not start with a virtual machine, and part of this topic's job is to make clear when one is and is not the right answer.

## What you will be able to do

- Read a VM size name and pick a suitable family for a workload.
- Design VM availability with zones and scale sets and state the SLA.
- Combine pay-as-you-go, reservations, savings plans and Spot sensibly.
- Choose disk types and encryption options.
- Build and distribute golden images.
- Explain confidential computing and when it is warranted.
- Decide whether to self-host a model on GPUs or use a managed model service, and what each needs.
- Recognise when Batch, HPC tooling, virtual desktops or specialised platforms fit.
- Choose between VMs, App Service, Functions, Container Apps and AKS for a given requirement.

## Prerequisites

[Topic 01](../01-cloud-and-azure-fundamentals/README.md) (regions, zones, quotas, SLAs), [Topic 02](../02-identity-and-access/README.md) (managed identities) and [Topic 05](../05-networking/README.md) (virtual networks, NSGs, Bastion).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Virtual Machines](01-virtual-machines.md) | 60 min | CLI |
| 02 | [Availability Sets, Zones and Virtual Machine Scale Sets](02-availability-sets-zones-and-virtual-machine-scale-sets.md) | 50 min | CLI |
| 03 | [Spot, Reserved Capacity and Compute Fleet](03-spot-reserved-capacity-and-compute-fleet.md) | 40 min | CLI |
| 04 | [Managed Disks](04-managed-disks.md) | 45 min | CLI |
| 05 | [Images and VM Image Builder](05-images-and-vm-image-builder.md) | 40 min | CLI |
| 06 | [Dedicated Host and Confidential Computing](06-dedicated-host-and-confidential-computing.md) | 40 min | – |
| 07 | [GPU and AI Infrastructure](07-gpu-and-ai-infrastructure.md) | 60 min | CLI |
| 08 | [Azure Batch](08-azure-batch.md) | 35 min | – |
| 09 | [High Performance Computing](09-high-performance-computing.md) | 30 min | – |
| 10 | [Azure Virtual Desktop and Windows 365](10-azure-virtual-desktop-and-windows-365.md) | 35 min | – |
| 11 | [Azure VMware Solution and Partner Clouds](11-azure-vmware-solution-and-partner-clouds.md) | 30 min | – |
| 12 | [Compute Decision Guide](12-compute-decision-guide.md) | 40 min | Exercise |

**AI engineers:** read 01, 03, 07 and 12, and skim the rest. **Architects:** all of it, with 02, 07 and 12 in depth.

## Completion checklist

- [ ] I can decode `Standard_D8ads_v6` and `Standard_NC24ads_A10_v4`.
- [ ] I can state the VM SLA for a single instance, an availability set and zones.
- [ ] I have created a scale set with autoscale across zones.
- [ ] I can explain the difference between a reservation, a savings plan and a capacity reservation.
- [ ] I can choose between Premium SSD, Premium SSD v2 and Ultra Disk.
- [ ] I have checked GPU quota in a region and know how to request more.
- [ ] I can list four ways to host an open model on Azure and when to pick each.
- [ ] I can justify a compute choice for a given workload using the decision guide.

## Maps to

- **AZ-900**: compute services.
- **AZ-104**: deploy and manage Azure compute resources.
- **AZ-305**: design compute solutions.
- **AI-103 / AI-300**: choose compute for model hosting and training.
- **AZ-140, AZ-120**: virtual desktop and SAP specialisations (awareness here).

## Next

[07 · Containers and Kubernetes](../07-containers-and-kubernetes/README.md)
