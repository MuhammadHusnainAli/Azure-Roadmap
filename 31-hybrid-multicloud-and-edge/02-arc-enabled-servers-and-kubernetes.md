# Arc-Enabled Servers and Kubernetes

## Why it matters

Once resources are projected into Azure (file 01), the value comes from operating them consistently: one patching process, one security posture, one monitoring pipeline and one way to deploy configuration, whether the server is in Azure, a datacentre or AWS, and whether the Kubernetes cluster is AKS, on Azure Local or at a factory. Arc-enabled servers and Arc-enabled Kubernetes bring Azure Policy, Defender for Cloud, Azure Monitor, Azure Update Manager, managed identity and GitOps to those environments. This file covers onboarding at scale and the day-2 operations architects design for hybrid estates.

## Key concepts

### Arc-enabled servers: capabilities

| Capability | Detail |
|---|---|
| **Governance** | Azure Policy, including **machine configuration** (audit/apply OS settings, formerly guest configuration); tags; RBAC |
| **Security** | Defender for Servers (Defender for Endpoint integration, vulnerability assessment), Microsoft Sentinel collection |
| **Monitoring** | Azure Monitor Agent with DCRs, VM insights, change tracking and inventory |
| **Updates** | **Azure Update Manager** schedules, assessments and maintenance configurations |
| **Access** | SSH via Arc and Windows Admin Center in the Azure portal without inbound ports; Entra login |
| **Identity** | System-assigned managed identity for apps to call Azure (Key Vault, Storage) |
| **Extensions** | Custom Script, Key Vault certificate sync, Run Command, Dependency agent, third-party |
| **Licensing** | **Extended Security Updates (ESU)** for older Windows Server delivered and billed through Arc; Windows Server pay-as-you-go |

Extension **allowlists/blocklists** and agent configuration (`azcmagent config`) control what can run.

### Arc-enabled Kubernetes: capabilities

| Capability | Detail |
|---|---|
| **GitOps with Flux v2** | Declarative configuration and app deployment from Git, at scale through Azure Policy |
| **Azure Policy for Kubernetes** | Gatekeeper-based admission policies (for example, no privileged containers) |
| **Monitoring** | Managed Prometheus and container logs (Topic 26 file 06) |
| **Defender for Containers** | Runtime threat detection and posture |
| **Cluster extensions** | Azure Key Vault Secrets Provider, Azure ML, Azure IoT Operations, Flux, Dapr, open service mesh alternatives, AI extensions |
| **Custom locations** | Expose the cluster as a deployment target for Azure services (data services, AI extensions) |
| **Cluster Connect** | Access the API server through Azure without inbound firewall rules |
| **Workload identity** | Entra workload identity for pods |

## How it works on Azure

GitOps configuration for an Arc cluster:

```bash
az k8s-configuration flux create -g rg-arc-k8s -c k8s-store-042 -t connectedClusters \
  -n store-baseline --namespace cluster-config --scope cluster \
  -u https://github.com/contoso/store-gitops --branch main \
  --kustomization name=infra path=./infrastructure prune=true \
  --kustomization name=apps path=./apps/store prune=true dependsOn=["infra"]
```

At scale, assign the built-in policy that deploys a Flux configuration to every Arc cluster with a given tag, so new stores get the baseline automatically.

Update Manager for Arc servers: create a maintenance configuration (for example second Tuesday, 02:00–04:00 local) with dynamic scope by tag `site=*` and `environment=prod`; enable periodic assessment via policy.

Machine configuration example: assign the built-in "Windows machines should meet requirements of the Azure compute security baseline" or Linux baseline initiative to Arc servers; review compliance per site.

Key Vault without secrets on an Arc server:

```bash
# On the server, a local app requests a token for Key Vault via the Arc managed identity endpoint (HIMDS),
# then calls Key Vault. Grant the Arc machine's identity "Key Vault Secrets User" on the vault.
az role assignment create --assignee <arc-machine-principal-id> --role "Key Vault Secrets User" --scope <vault-id>
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Onboarding method** | Scripted for small estates; Configuration Manager/GPO/Ansible for Windows/Linux fleets; VMware vSphere/SCVMM integration to onboard VMs at scale. |
| **Policy model** | Same management group policies as Azure where possible; site-specific exceptions documented. |
| **Patching** | Azure Update Manager centrally for Azure and Arc servers; align maintenance windows to site operations. |
| **GitOps structure** | Separate infrastructure and app repos/paths; per-site overlays with Kustomize. |
| **Extension governance** | Allowlist approved extensions; restrict Run Command. |
| **Disconnected sites** | Plan for intermittent connectivity; Arc caches some state, but management needs connectivity (consider Azure Local disconnected operations, file 04). |

## Hands-on

1. Onboard two servers; deploy AMA, Defender for Servers and a machine configuration baseline via policy.
2. Create an Update Manager maintenance configuration with dynamic scope; run an assessment.
3. Configure an extension allowlist on a server and test blocking an extension.
4. Connect a Kubernetes cluster; apply a Flux configuration with infra and app kustomizations.
5. Assign Azure Policy for Kubernetes to block privileged pods; test a violating deployment.

## Security, cost and reliability notes

- **Least privilege**: "Azure Connected Machine Onboarding" role for onboarding only; separate roles for extension management.
- **ESU and PAYG licensing** add costs; track per site.
- **GitOps security**: protect repositories; use signed commits and private repos; Flux pulls, so no cluster credentials in CI.
- **Agent updates**: keep the Connected Machine agent and Arc K8s agents current.

## Common mistakes

| Mistake | Fix |
|---|---|
| Different patching tools per environment | Update Manager for all |
| Manual configuration on edge clusters | GitOps with policy-driven deployment |
| Unrestricted extensions and Run Command | Allowlists and RBAC |
| Broad onboarding service principals left active | Scoped, time-bound onboarding credentials |
| Stale agents | Automatic agent upgrades |

## Check yourself

1. List six capabilities of Arc-enabled servers.
2. What is machine configuration?
3. How does GitOps work on Arc-enabled Kubernetes, and how do you apply it at scale?
4. What are cluster extensions and Cluster Connect?
5. How do Arc servers access Azure services without secrets?
6. How are ESUs delivered for older Windows Server?

<details>
<summary>Answers</summary>

1. Any six: Azure Policy/machine configuration, Defender for Servers, AMA monitoring, Update Manager, SSH/Windows Admin Center access, managed identity, extensions, ESU and PAYG licensing.
2. Azure Policy capability that audits or applies OS-level settings inside machines (formerly guest configuration).
3. Flux v2 pulls configuration from Git into the cluster; apply at scale with Azure Policy deploying Flux configurations to tagged clusters.
4. Extensions deploy Azure capabilities into clusters; Cluster Connect provides API server access through Azure without inbound ports.
5. Through the Arc system-assigned managed identity and Azure RBAC.
6. Through Azure Arc, enabled and billed per server in Azure.

</details>

## References

- [Arc-enabled servers overview](https://learn.microsoft.com/azure/azure-arc/servers/overview)
- [GitOps with Flux v2](https://learn.microsoft.com/azure/azure-arc/kubernetes/conceptual-gitops-flux2)
- [Arc-enabled Kubernetes overview](https://learn.microsoft.com/azure/azure-arc/kubernetes/overview)
- [Azure Update Manager](https://learn.microsoft.com/azure/update-manager/overview)
- [Machine configuration](https://learn.microsoft.com/azure/governance/machine-configuration/overview)
