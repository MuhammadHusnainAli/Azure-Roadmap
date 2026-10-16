# AKS Fleet Manager and GitOps

> Topic: 07 · Containers and Kubernetes · Tracks: E ○ A ● C – · Status checked: 2026-10

## Why it matters

Organisations rarely end with one cluster. They have clusters per environment, per region, per business unit and per compliance zone. Upgrading twenty clusters by hand, or keeping their configuration consistent, does not scale. **Azure Kubernetes Fleet Manager** orchestrates upgrades and workload placement across clusters, and **GitOps** makes Git the source of truth for what runs in each cluster. Together they turn a cluster estate into something a small platform team can operate.

## Key concepts

### Azure Kubernetes Fleet Manager

A **fleet** is an Azure resource that groups **member clusters** (AKS, and Arc-enabled Kubernetes for some features). Members can be in different subscriptions and regions within the same tenant.

| Capability | What it does |
|---|---|
| **Update orchestration** | Upgrade Kubernetes versions and node images across members in a controlled order |
| **Auto-upgrade profiles** | Trigger fleet-wide update runs automatically when new versions appear on a channel |
| **Resource propagation (placement)** | Define Kubernetes resources once on a hub cluster and place them onto selected members |
| **Multi-cluster load balancing** | L4 load balancing across member clusters for services (check current status) |
| **Managed namespaces** | Consistent namespace, quota and RBAC across members (check current status) |

Fleets can be created **without a hub** (update orchestration only) or **with a hub cluster** (a Microsoft-managed AKS cluster that hosts placement APIs).

### Update runs, stages and groups

```text
Update strategy
  Stage 1: "canary"   groups: [dev-weu]                       wait 1 day after
  Stage 2: "early"    groups: [test-weu, test-neu]            wait 1 day after
  Stage 3: "prod"     groups: [prod-weu] then [prod-neu]      (groups in a stage run in parallel)

Update run = strategy + target (Kubernetes version or node image) + upgrade type
```

- Members are assigned to **update groups**.
- A **strategy** orders groups into **stages** with waits between them.
- An **update run** applies a target version through the strategy; it can be paused, stopped, or skipped for specific members.
- Runs honour each cluster's planned maintenance windows.
- With **approval gates** (check current status), a human or automation approves progression between stages.

### Resource placement

On the hub, you create normal Kubernetes resources (namespaces, deployments, config) and a **ClusterResourcePlacement** (or namespace-scoped `ResourcePlacement`) that selects resources and member clusters:

| Policy | Behaviour |
|---|---|
| `PickAll` | Every member that matches label selectors |
| `PickN` | N clusters chosen by affinity, topology spread or properties (for example available CPU, cost) |
| `PickFixed` | Named clusters |

Rollout strategies, overrides per cluster (for example a region-specific setting), and drift detection are supported. Placement is useful for platform components that must exist everywhere; for application delivery, many teams use GitOps instead.

### GitOps

GitOps applies desired state from Git to clusters with an **in-cluster agent that pulls** changes, rather than a pipeline that pushes with cluster credentials.

| Principle | Effect |
|---|---|
| Declarative | Everything as manifests, Helm charts or Kustomize overlays |
| Versioned and immutable | Git history is the audit log; rollback is `git revert` |
| Pulled automatically | Agents reconcile continuously |
| Continuously reconciled | Drift (manual `kubectl` changes) is reverted |

### Flux and Argo CD on AKS

| | **Flux v2** | **Argo CD** |
|---|---|---|
| Azure integration | **Microsoft.Flux cluster extension**, managed via `az k8s-configuration flux`, Azure Policy at scale, works on AKS and Arc | **Argo CD cluster extension** (check current status), or self-installed |
| Model | Set of controllers (source, kustomize, helm, notification, image automation) | Server with UI, application CRDs, ApplicationSets |
| UI | None built in (use Azure portal views or third-party UIs) | Rich web UI |
| Multi-tenancy | Namespaced, per-team configurations | Projects and RBAC |
| Fit | Azure-native, policy-driven fleet bootstrap | Teams wanting a UI and app-centric workflows |

A common split: Flux (via the Azure extension and policy) bootstraps platform components on every cluster; teams use Flux or Argo CD for their applications.

### Repository layout

```text
platform-gitops/
├─ clusters/
│   ├─ dev-weu/          kustomization.yaml → base + dev overlays
│   ├─ prod-weu/
│   └─ prod-neu/
├─ infrastructure/       ingress, cert-manager, policies, monitoring (Helm releases)
└─ apps/
    ├─ base/chat-api/    deployment, service, hpa
    └─ overlays/
        ├─ dev/          replicas: 1, image tag from dev
        └─ prod/         replicas: 3, image digest pinned
```

Image promotion: CI builds and pushes an image, then opens a pull request updating the tag or digest in the overlay. Merge equals deploy.

## How it works on Azure

```bash
# Fleet without a hub, three members, one update strategy
az extension add -n fleet --upgrade -y
az fleet create -g $RG -n fleet-main -l $LOC

az fleet member create -g $RG -f fleet-main -n dev-weu  --member-cluster-id $DEV_ID  --update-group canary
az fleet member create -g $RG -f fleet-main -n prod-weu --member-cluster-id $PRODW_ID --update-group prod
az fleet member create -g $RG -f fleet-main -n prod-neu --member-cluster-id $PRODN_ID --update-group prod

cat > strategy.json <<'EOF'
{ "stages": [
  { "name": "canary", "groups": [ { "name": "canary" } ], "afterStageWaitInSeconds": 86400 },
  { "name": "prod",   "groups": [ { "name": "prod" } ] } ] }
EOF
az fleet updatestrategy create -g $RG -f fleet-main -n safe-rollout --stages strategy.json

az fleet updaterun create -g $RG -f fleet-main -n upgrade-1-xx \
  --upgrade-type Full --kubernetes-version <target-version> --update-strategy-name safe-rollout
az fleet updaterun start -g $RG -f fleet-main -n upgrade-1-xx
az fleet updaterun show  -g $RG -f fleet-main -n upgrade-1-xx -o table
```

```bash
# Flux GitOps configuration on a cluster
az k8s-extension create -g $RG -c $AKS -t managedClusters --name flux --extension-type microsoft.flux
az k8s-configuration flux create -g $RG -c $AKS -t managedClusters -n platform \
  --namespace flux-system --scope cluster \
  -u https://github.com/<org>/platform-gitops --branch main \
  --kustomization name=infra path=./infrastructure prune=true \
  --kustomization name=apps path=./clusters/prod-weu prune=true dependsOn=["infra"]
az k8s-configuration flux show -g $RG -c $AKS -t managedClusters -n platform --query complianceState
```

Use Azure Policy's built-in definitions to require a Flux configuration on every cluster in a management group.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Fleet for upgrades | Yes, once you have more than a handful of clusters; order dev → test → prod with soak time |
| Hub cluster | Only if you need resource placement or multi-cluster load balancing |
| Placement vs GitOps | Placement for uniform platform resources from Azure; GitOps for application delivery with PR-based change control |
| Flux vs Argo CD | Flux for Azure-native, policy-enforced bootstrap; Argo CD where teams want its UI; avoid running both for the same resources |
| Repository structure | Separate platform and application repositories; overlays per environment |
| Secrets in GitOps | Never plain secrets in Git. Use Key Vault CSI driver, External Secrets Operator, or sealed secrets |
| Image references | Digests in production overlays |

## Hands-on

1. Create two small AKS clusters (or reuse earlier labs) and a fleet without a hub.
2. Join both, with one in a `canary` group and one in a `prod` group.
3. Create a strategy with a short wait (for example 10 minutes) and run a **node image** update (`--upgrade-type NodeImageOnly`), which is low risk. Watch the stages progress.
4. Install the Flux extension on one cluster, fork a sample GitOps repository (for example the Flux `podinfo` example), and point a configuration at it.
5. Delete a deployment with `kubectl` and watch Flux recreate it. Then change the replica count in Git and watch it reconcile.

## Security, cost and reliability notes

- **Security:** GitOps removes the need for CI pipelines to hold cluster admin credentials; protect the Git repository (branch protection, required reviews, signed commits) because it now controls production.
- **Security:** limit who can create update runs and placements on the fleet with Azure RBAC.
- **Cost:** fleets without a hub have no extra cluster cost; a hub cluster is billed.
- **Reliability:** staged upgrades with soak time catch issues before production. Combine with PodDisruptionBudgets on every workload.
- **Reliability:** reconcile loops will undo emergency manual fixes. Make hotfixes through Git, or suspend reconciliation explicitly and document it.

## Common mistakes

- Upgrading all clusters at once, or production first.
- Manual `kubectl` changes in GitOps-managed clusters, silently reverted.
- Secrets committed to the GitOps repository.
- Both Flux and Argo CD managing the same objects.
- A single repository where every team can change every cluster.
- Letting an update run continue to production without checking the canary stage.

## Check yourself

1. What are update groups, stages and update runs in Fleet Manager?
2. When do you need a fleet hub cluster?
3. What are the three placement policy types?
4. Why is GitOps described as "pull-based", and what security benefit does that bring?
5. How do Flux and Argo CD differ in their Azure integration?
6. How should secrets be handled in a GitOps workflow?
7. What happens to a manual change in a Flux-managed cluster?

<details>
<summary>Answers</summary>

1. Groups are sets of member clusters; stages order groups with optional waits; an update run applies a target version or node image through a strategy of stages.
2. When you use resource propagation (placement) or multi-cluster load balancing; update orchestration alone does not need one.
3. `PickAll`, `PickN` and `PickFixed`.
4. An agent in the cluster pulls desired state from Git and applies it; pipelines no longer need credentials to push into clusters, reducing exposure.
5. Flux is available as a Microsoft-managed cluster extension, configurable with Azure CLI and enforceable with Azure Policy across AKS and Arc; Argo CD is offered as an extension more recently or self-installed, and brings a rich UI.
6. Keep secrets out of Git: reference Key Vault through the CSI driver or External Secrets Operator, or use encrypted sealed secrets.
7. It is detected as drift and reverted to the state in Git at the next reconciliation.

</details>

## References

- [Azure Kubernetes Fleet Manager overview](https://learn.microsoft.com/azure/kubernetes-fleet/overview)
- [Update orchestration](https://learn.microsoft.com/azure/kubernetes-fleet/update-orchestration)
- [Auto-upgrade profiles](https://learn.microsoft.com/azure/kubernetes-fleet/update-automation)
- [Resource propagation](https://learn.microsoft.com/azure/kubernetes-fleet/concepts-resource-propagation)
- [GitOps with Flux v2 on AKS and Arc](https://learn.microsoft.com/azure/azure-arc/kubernetes/conceptual-gitops-flux2)
- [Argo CD extension](https://learn.microsoft.com/azure/azure-arc/kubernetes/tutorial-use-gitops-argocd)
- [GitOps for AKS (Architecture Center)](https://learn.microsoft.com/azure/architecture/example-scenario/gitops-aks/gitops-blueprint-aks)
- [Flux documentation](https://fluxcd.io/flux/)
