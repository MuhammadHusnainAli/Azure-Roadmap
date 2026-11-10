# Azure Red Hat OpenShift and Service Fabric

## Why it matters

Not every customer arrives with a clean slate. Many enterprises already run Red Hat OpenShift on-premises and want the same platform in Azure. Others have years of applications on Service Fabric, Microsoft's older microservices platform. Architects and consultants must recognise when these platforms are the right answer, what they cost, and how to plan a move from them when they are not.

## Key concepts

### Azure Red Hat OpenShift (ARO)

ARO is a fully managed **OpenShift** service, **jointly engineered, operated and supported by Microsoft and Red Hat**. OpenShift is a Kubernetes distribution with an opinionated platform on top.

| What OpenShift adds over plain Kubernetes | Detail |
|---|---|
| Developer experience | Web console, Source-to-Image builds, templates, integrated pipelines (Tekton) and GitOps (Argo CD) |
| Security defaults | Security Context Constraints (stricter than default Kubernetes), integrated OAuth |
| Operators | OperatorHub with certified operators |
| Networking | OVN-Kubernetes, Routes (in addition to Ingress) |
| Service mesh, serverless, virtualisation | OpenShift Service Mesh, Serverless (Knative), OpenShift Virtualization |
| AI | OpenShift AI for model serving and MLOps |
| Consistency | Same platform on-premises, in Azure and other clouds |

ARO specifics:

| Item | Detail |
|---|---|
| Responsibility | Microsoft and Red Hat run control plane and infrastructure nodes, patching and upgrades in a shared model; you run your workloads |
| Billing | Azure VM costs plus an OpenShift licence charge on worker nodes, billed by Azure; reserved instances available for the licence portion |
| Support | Single support path through Azure, with Red Hat engaged jointly |
| Minimum footprint | Three control plane and three worker nodes; larger than a minimal AKS cluster |
| Networking | Deployed into your VNet; public or private API and ingress |
| Identity | Integrates with Microsoft Entra ID through OpenShift OAuth; managed identities for platform components (check current status) |
| Compliance | Broad Azure compliance coverage, with SLA |

**When ARO fits**

- The organisation is standardised on OpenShift and wants consistent tooling and skills across on-premises and Azure.
- Workloads depend on OpenShift-specific features (Routes, SCCs, certified operators, OpenShift Virtualization, OpenShift AI).
- The customer values a jointly supported, opinionated platform over assembling AKS add-ons.

**When AKS fits better**

- No existing OpenShift investment; lower minimum cost; tighter integration with Azure-native features (KAITO, AKS Automatic, Fleet Manager, Container Apps neighbours).

### Azure Service Fabric

Service Fabric is a distributed systems platform for packaging, deploying and managing microservices, including **stateful** services with replicated state co-located with code. Microsoft built large parts of Azure itself on it (Azure SQL Database, Cosmos DB, Event Hubs and others historically used Service Fabric internally).

| Concept | Detail |
|---|---|
| Programming models | Reliable Services (stateless and stateful), Reliable Actors, guest executables, containers |
| Stateful services | Reliable Collections replicated across nodes; low latency state with no external database |
| Clusters | **Service Fabric managed clusters** (recommended: Azure manages the underlying resources) or classic clusters on scale sets you manage |
| OS | Windows (most common) and Linux |
| Tooling | Visual Studio, Service Fabric Explorer, PowerShell |

Status: Service Fabric remains supported, with managed clusters the recommended deployment model. It receives far less ecosystem investment than Kubernetes, the skills pool is shrinking, and new greenfield microservices on Azure generally target Container Apps or AKS.

**When to keep Service Fabric**

- Large, stable estates where stateful Reliable Services or Actors are deeply used and working well.
- Windows-heavy .NET Framework services where migration cost outweighs benefit.

**When to plan a move**

- New feature work is slowing because of platform skills or tooling.
- Services are mostly stateless or guest executables, which map directly to containers.
- The organisation is standardising on Kubernetes or Container Apps.

## How it works on Azure

### Migration considerations

**From Service Fabric**

| Service Fabric construct | Target |
|---|---|
| Stateless Reliable Service, guest executable | Container on Container Apps or AKS |
| Stateful Reliable Service | Stateless service plus external state (Cosmos DB, Azure SQL, Managed Redis), or Dapr state |
| Reliable Actors | Dapr actors (on Container Apps or AKS), or Orleans |
| Service Fabric naming and communication | Kubernetes Services, Dapr service invocation |
| Application and service manifests | Kubernetes manifests, Helm, Container Apps definitions |
| Windows .NET Framework services | Windows containers on AKS, or modernise to .NET on Linux |

The hardest part is almost always **state**: moving from co-located replicated state to an external store changes latency, consistency and cost. Prototype the stateful paths first.

**From OpenShift elsewhere to Azure**

| Path | When |
|---|---|
| OpenShift on-premises → **ARO** | Keep the platform; least change; use Migration Toolkit for Containers |
| OpenShift → **AKS** | Reduce licence cost or adopt Azure-native features; replace Routes, SCCs, BuildConfigs and OpenShift-specific operators; retrain teams |
| Self-managed OpenShift on Azure VMs → ARO | Offload platform operations |

**From AKS or Kubernetes to ARO** is rarer; it happens when an enterprise chooses OpenShift as its strategic platform.

### Decision summary

| Situation | Recommendation |
|---|---|
| Greenfield microservices, no platform constraint | Container Apps, then AKS |
| Kubernetes standard, Azure-first | AKS |
| OpenShift standard, hybrid estate | ARO |
| Large working Service Fabric estate | Keep on managed clusters; modernise incrementally |
| Small or stateless Service Fabric apps | Migrate to Container Apps or AKS |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| ARO vs AKS cost | Compare total cost including OpenShift licence and the larger minimum footprint against AKS plus the add-ons and tooling you would assemble |
| Skills | ARO preserves OpenShift skills; AKS builds Kubernetes and Azure skills more broadly |
| Service Fabric state | Decide early whether to externalise state or adopt Dapr actors |
| Big-bang vs incremental | Incremental with a strangler pattern (topic 10 file 11): new services on the target, old services retired one by one |

## Hands-on

ARO clusters are relatively expensive to run, so this hands-on is mainly analysis.

1. Use the Azure pricing calculator to compare: an ARO cluster with three workers of `Standard_D8s_v5`, and an AKS Standard cluster with the same workers. Note the licence component.
2. List the OpenShift-specific features a sample workload uses (Routes, SCCs, BuildConfigs, ImageStreams, certified operators), and for each, write the AKS equivalent or replacement.
3. For a Service Fabric application you know (or the public "Voting" sample), classify each service as stateless, stateful or actor, and map it to a target from the table above.

```bash
# If you want to see ARO's prerequisites (registering providers, quota for the minimum footprint)
az provider register -n Microsoft.RedHatOpenShift --wait
az vm list-usage -l swedencentral --query "[?contains(name.value,'standardDSv5Family')]" -o table
```

## Security, cost and reliability notes

- **Security:** ARO's SCCs and integrated OAuth are strict by default; workloads built for permissive Kubernetes may need changes.
- **Security:** Service Fabric clusters need certificate management for node and client authentication; managed clusters simplify it.
- **Cost:** ARO's minimum footprint and licence make small clusters comparatively expensive. Consolidate workloads.
- **Reliability:** both ARO and Service Fabric managed clusters support availability zones; design across zones from the start.

## Common mistakes

- Choosing ARO without an OpenShift requirement, paying licence costs for features nobody uses.
- Moving from OpenShift to AKS assuming manifests will work unchanged.
- Migrating stateful Service Fabric services without redesigning state.
- Starting new greenfield applications on Service Fabric.
- Underestimating retraining effort in either direction.

## Check yourself

1. Who operates and supports Azure Red Hat OpenShift?
2. Name four things OpenShift adds over plain Kubernetes.
3. When is ARO preferable to AKS?
4. What makes Service Fabric stateful services hard to migrate?
5. What can replace Reliable Actors on Kubernetes or Container Apps?
6. What is the recommended deployment model for Service Fabric today?

<details>
<summary>Answers</summary>

1. Microsoft and Red Hat jointly, with a single support path through Azure.
2. Any four of: web console and developer tooling, Source-to-Image builds, Routes, Security Context Constraints, integrated OAuth, OperatorHub with certified operators, integrated pipelines and GitOps, OpenShift Virtualization, OpenShift AI.
3. When the organisation is standardised on OpenShift, needs consistency with on-premises OpenShift, or depends on OpenShift-specific features.
4. State lives in replicated Reliable Collections co-located with the code; moving it to an external store changes latency, consistency, transactions and cost.
5. Dapr actors, or Microsoft Orleans.
6. Service Fabric managed clusters.

</details>

## References

- [Azure Red Hat OpenShift documentation](https://learn.microsoft.com/azure/openshift/)
- [ARO support policy and responsibility matrix](https://learn.microsoft.com/azure/openshift/responsibility-matrix)
- [Choose an Azure container service](https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service)
- [Service Fabric overview](https://learn.microsoft.com/azure/service-fabric/service-fabric-overview)
- [Service Fabric managed clusters](https://learn.microsoft.com/azure/service-fabric/overview-managed-cluster)
- [Migrate from Service Fabric to AKS](https://learn.microsoft.com/azure/service-fabric/service-fabric-migrate-to-aks)
- [Dapr actors](https://docs.dapr.io/developing-applications/building-blocks/actors/actors-overview/)
