# 07 · Containers and Kubernetes

> Tracks: E ● A ● C ○ · Part B · Core Infrastructure and Application Platform

Containers are now the default packaging format for cloud applications, including AI applications, agents and MCP servers. Azure offers a ladder of container hosts, from a single container started in seconds (Azure Container Instances), through a serverless application platform (Azure Container Apps), to full Kubernetes (Azure Kubernetes Service). This topic covers the ladder, the registry behind it, and the specific features that matter for AI: serverless GPUs, code-interpreter sandboxes, GPU node pools and KAITO.

## What you will be able to do

- Build small, secure, multi-stage container images and explain how layers and registries work.
- Run a private Azure Container Registry with geo-replication, automated builds and private access.
- Choose between Container Instances, Container Apps and AKS for a given workload, and defend the choice.
- Deploy and scale an app on Container Apps with revisions, KEDA rules, Dapr and jobs.
- Host an agent, an MCP server or a GPU inference workload on Container Apps.
- Create an AKS cluster (Automatic or Standard), and plan its networking, identity, upgrades and cost.
- Run open models on AKS with GPU node pools and KAITO.
- Manage many clusters with Fleet Manager and GitOps.

## Prerequisites

[Topic 02](../02-identity-and-access/README.md) (managed identities, Workload Identity), [Topic 04](../04-tooling-and-infrastructure-as-code/README.md) (CLI, Bicep), [Topic 05](../05-networking/README.md) (VNets, private endpoints, load balancing) and [Topic 06](../06-compute/README.md) (VM sizes, GPU infrastructure). You should be able to write a Dockerfile and run `docker build` locally.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Container Fundamentals](01-container-fundamentals.md) | 45 min | Docker |
| 02 | [Azure Container Registry](02-azure-container-registry.md) | 45 min | CLI |
| 03 | [Azure Container Instances](03-azure-container-instances.md) | 30 min | CLI |
| 04 | [Azure Container Apps](04-azure-container-apps.md) | 60 min | CLI |
| 05 | [Container Apps for AI](05-container-apps-for-ai.md) | 50 min | CLI, Python |
| 06 | [AKS Fundamentals](06-aks-fundamentals.md) | 60 min | CLI |
| 07 | [AKS Networking](07-aks-networking.md) | 60 min | CLI |
| 08 | [AKS Security and Identity](08-aks-security-and-identity.md) | 50 min | CLI, YAML |
| 09 | [AKS Scaling and Cost](09-aks-scaling-and-cost.md) | 45 min | CLI, YAML |
| 10 | [AKS for AI Workloads](10-aks-for-ai-workloads.md) | 50 min | CLI, YAML |
| 11 | [AKS Fleet Manager and GitOps](11-aks-fleet-manager-and-gitops.md) | 40 min | CLI |
| 12 | [Azure Red Hat OpenShift and Service Fabric](12-azure-red-hat-openshift-and-service-fabric.md) | 25 min | – |
| 13 | [Azure Container Storage](13-azure-container-storage.md) | 30 min | YAML |

**AI engineers:** 01, 02, 04 and 05 in depth, then 06, 08 and 10. **Architects:** all of it, with 04, 06, 07 and 08 in depth. **Consultants:** 01, 03, 04, 06 and 12 for positioning.

## Completion checklist

- [ ] I can write a multi-stage Dockerfile that produces a small, non-root image.
- [ ] I have pushed to and pulled from ACR using a managed identity, with no admin user.
- [ ] I can state which ACR tier is needed for geo-replication and private endpoints.
- [ ] I can explain the difference between a Container Apps revision, replica and job.
- [ ] I have scaled a container app to zero and back on an HTTP or queue rule.
- [ ] I can explain what dynamic sessions are for and why they matter for agents.
- [ ] I can choose between AKS Automatic and Standard and justify it.
- [ ] I can explain Azure CNI Overlay and why it is the default.
- [ ] I have configured Workload Identity for a pod calling an Azure service.
- [ ] I can describe how KAITO deploys an open model.
- [ ] I can explain how Fleet Manager orchestrates upgrades across clusters.

## Maps to

- **AZ-104**: deploy and manage containers (ACR, ACI, Container Apps), AKS basics.
- **AI-200**: implement containerised solutions.
- **AZ-305**: design compute solutions including containers.
- **AI-103**: host AI apps and agents.
- **CKA / CKAD** (CNCF): general Kubernetes skills beyond Azure.

## Next

[08 · Storage](../08-storage/README.md)
