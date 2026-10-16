# AKS Fundamentals

> Topic: 07 · Containers and Kubernetes · Tracks: E ○ A ● C ○ · Status checked: 2026-10

## Why it matters

Azure Kubernetes Service (AKS) is Azure's managed Kubernetes. It is the platform of choice for organisations standardising on Kubernetes, running many teams' workloads on shared infrastructure, or needing Kubernetes-native tooling such as operators, service meshes and GitOps. It is also the most operationally demanding container option. Architects must be able to say when AKS is warranted and, when it is, design it so that upgrades, security and cost stay under control.

## Key concepts

### Kubernetes in one table

| Object | Purpose |
|---|---|
| **Pod** | One or more containers scheduled together; the smallest unit |
| **Deployment** | Desired number of identical pods, rolling updates |
| **StatefulSet** | Pods with stable identity and storage |
| **DaemonSet** | One pod per node (agents, log shippers) |
| **Job / CronJob** | Run to completion, on a schedule |
| **Service** | Stable virtual IP and DNS name for a set of pods |
| **Ingress / Gateway** | HTTP routing from outside the cluster |
| **ConfigMap / Secret** | Configuration and secrets |
| **Namespace** | Logical partition for teams, quotas, RBAC |
| **PersistentVolumeClaim** | Request for storage |
| **Custom Resource (CRD)** | Extension types managed by operators |

### What AKS manages and what you manage

```text
Microsoft-managed control plane (free or paid tier)
  API server · etcd · scheduler · controller manager · cloud controller
───────────────────────────────────────────────────────────────────────
Your subscription (node resource group MC_*)
  Node pools = Virtual Machine Scale Sets (or VMs)  ← you choose size, count, OS, zones
  kubelet · container runtime (containerd) · kube-proxy or Cilium · add-ons
  Load balancers, public IPs, disks, NSG, route tables created by AKS
```

You manage node pools, workloads, upgrades (or their automation), add-ons and configuration. Microsoft runs the control plane and provides node images.

### Cluster tiers

| Tier | Control plane SLA | Notes |
|---|---|---|
| **Free** | None (SLO only) | Dev and test; limited scale |
| **Standard** | 99.95% with zones (99.9% without) | Production; larger scale limits |
| **Premium** | As Standard | Adds **Long-Term Support** (two-year support for selected Kubernetes versions) |

### AKS Automatic vs AKS Standard

| | **AKS Automatic** | **AKS Standard** |
|---|---|---|
| Philosophy | Production-ready defaults, Microsoft manages nodes | Full control, you configure everything |
| Nodes | Managed with node auto-provisioning (Karpenter-based); no node pools to size | Node pools you size and scale |
| Networking | Azure CNI Overlay with Cilium; managed ingress; managed NAT | You choose CNI, ingress, egress |
| Security defaults | Entra auth with Azure RBAC, Workload Identity, deployment safeguards, image cleaner, no SSH to nodes | Opt in to each |
| Scaling | HPA, KEDA, VPA enabled | Opt in |
| Upgrades | Automatic cluster and node OS upgrades in maintenance windows | You choose channels |
| Monitoring | Managed Prometheus, Container Insights, Grafana integration | Opt in |
| Flexibility | Some settings locked | Anything Kubernetes allows |
| Fit | Most new application clusters, teams new to Kubernetes | Specialised needs, existing platforms, unusual networking |

Start with **Automatic** unless you can name the Standard feature you need.

### Node pools

- **System node pool**: runs CoreDNS, metrics server and other critical pods. At least one is required; use three nodes across zones for production, and taint it so application pods stay off (`CriticalAddonsOnly=true:NoSchedule`).
- **User node pools**: run applications. Separate pools for different VM sizes (general, memory-heavy, GPU, Arm), OS (Linux, Windows), Spot, or isolation needs.
- Node OS: **Azure Linux** (lean, Microsoft-maintained) or Ubuntu for Linux; Windows Server for Windows containers.
- Each pool is a scale set. Use zones (`--zones 1 2 3`) at creation; it cannot be added later.

### Kubernetes versions and the support policy

| Rule | Detail |
|---|---|
| Supported minor versions | AKS supports roughly three GA minor versions (N to N-2) at a time |
| Release cadence | Upstream releases about three minor versions a year; each is supported in AKS for about a year |
| Platform support | Clusters outside support get limited help: Microsoft only supports platform components |
| LTS | Premium tier, selected versions, about two years |
| Upgrades | One minor version at a time for the control plane; nodes follow |

An unpatched AKS cluster becomes unsupported within a year. Upgrades are not optional.

### Upgrades

| Layer | Mechanism |
|---|---|
| **Kubernetes version** | Auto-upgrade channel: `none`, `patch`, `stable`, `rapid`, `node-image` |
| **Node OS image** | Node OS upgrade channel: `None`, `Unmanaged`, `SecurityPatch`, `NodeImage` (weekly images) |
| **When** | Planned maintenance windows (`aksManagedAutoUpgradeSchedule`, `aksManagedNodeOSUpgradeSchedule`) |
| **How** | Surge upgrade: new nodes added (`max-surge`, for example 33%), old nodes cordoned and drained, respecting **PodDisruptionBudgets** |
| **Safety** | Upgrade non-production first; for fleets use Fleet Manager update runs (file 11); blue-green node pools for risky changes |

Workloads survive upgrades only if they run multiple replicas, have PodDisruptionBudgets, readiness probes and graceful shutdown.

## How it works on Azure

### Resource layout

- Cluster resource in your resource group; node infrastructure in a separate **node resource group** (`MC_<rg>_<cluster>_<region>` by default; name it explicitly). Lock it down with node resource group lockdown so users cannot change AKS-managed resources directly.
- Cluster identity: a managed identity used by AKS to manage Azure resources; kubelet identity used by nodes (for ACR pulls).

### Add-ons and extensions worth knowing

| Feature | Purpose |
|---|---|
| Azure Monitor (managed Prometheus, Container Insights) | Metrics, logs, dashboards |
| Azure Policy add-on | Gatekeeper-based guardrails |
| Key Vault Secrets Store CSI driver | Mount Key Vault secrets |
| Application routing (managed ingress) | Managed ingress controller and DNS integration (file 07) |
| Istio-based service mesh add-on | Managed Istio |
| KEDA, VPA | Event-driven and vertical scaling |
| Workload Identity | Pod identities to Entra |
| KAITO | AI model deployment (file 10) |
| Flux GitOps extension, Dapr extension, Backup extension | Cluster extensions |

### Cluster creation essentials

```bash
# Automatic
az aks create -g $RG -n aks-auto --sku automatic --location $LOC

# Standard, production-shaped
az aks create -g $RG -n aks-std --tier standard \
  --kubernetes-version <supported-version> \
  --node-count 3 --zones 1 2 3 --node-vm-size Standard_D4ds_v5 --os-sku AzureLinux \
  --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium \
  --enable-aad --enable-azure-rbac --disable-local-accounts \
  --enable-oidc-issuer --enable-workload-identity \
  --auto-upgrade-channel patch --node-os-upgrade-channel NodeImage \
  --enable-managed-identity --generate-ssh-keys
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| AKS at all | Use when Kubernetes itself is a requirement or a strategic platform; otherwise Container Apps (file 04) |
| Automatic or Standard | Automatic by default; Standard for specific control needs |
| Tier | Standard for production; Premium only if LTS is needed; Free for labs |
| How many clusters | Few, shared clusters with namespaces for teams reduce cost and toil; separate clusters for hard isolation, different compliance, or blast-radius limits. Always separate prod and non-prod |
| Node pool layout | Dedicated system pool; user pools by workload shape; avoid many tiny pools |
| Node OS | Azure Linux for smaller footprint and faster patching unless you depend on Ubuntu packages |
| Upgrade strategy | Automatic channels with maintenance windows, plus PDBs and multi-replica workloads |
| Multi-region | Independent clusters per region behind Front Door; do not stretch a cluster across regions |

## Hands-on

```bash
RG=rg-aks-lab; LOC=swedencentral; AKS=aks-lab
az group create -n $RG -l $LOC -o none

# 1. Create an AKS Automatic cluster (takes several minutes)
az aks create -g $RG -n $AKS --sku automatic -l $LOC -o none
az aks get-credentials -g $RG -n $AKS --overwrite-existing

# kubectl and kubelogin are needed for Entra auth
az aks install-cli
kubectl get nodes -o wide       # nodes appear once workloads need them (node auto-provisioning)

# 2. Deploy a workload with replicas, probes and a PodDisruptionBudget
kubectl create namespace demo
cat <<'EOF' | kubectl apply -n demo -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: hello }
spec:
  replicas: 3
  selector: { matchLabels: { app: hello } }
  template:
    metadata: { labels: { app: hello } }
    spec:
      containers:
      - name: hello
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports: [ { containerPort: 80 } ]
        resources:
          requests: { cpu: 100m, memory: 128Mi }
          limits:   { memory: 256Mi }
        readinessProbe: { httpGet: { path: /, port: 80 } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: hello-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: hello } }
---
apiVersion: v1
kind: Service
metadata: { name: hello }
spec:
  selector: { app: hello }
  ports: [ { port: 80, targetPort: 80 } ]
EOF
kubectl get pods,svc,pdb -n demo

# 3. Inspect versions and upgrade settings
az aks get-upgrades -g $RG -n $AKS -o table
az aks show -g $RG -n $AKS --query "{k8s:kubernetesVersion, tier:sku.tier, autoUpgrade:autoUpgradeProfile}" -o json

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** Entra integration with Azure RBAC, local accounts disabled, private or IP-restricted API server, Workload Identity, policy guardrails (file 08).
- **Cost:** the Free tier control plane is free; Standard and Premium charge per cluster hour. Nodes are the main cost; right-size requests and use autoscaling (file 09).
- **Reliability:** zones for every node pool, three-node system pool, multiple replicas with topology spread constraints, PDBs, readiness probes.
- **Reliability:** unsupported versions get no fixes. Track the AKS release calendar and automate upgrades.

## Common mistakes

- Choosing AKS for a single small API.
- One giant cluster for prod and non-prod together.
- Single-replica deployments with a PDB of `minAvailable: 1`, which blocks node drains and stalls upgrades.
- No resource requests, so the scheduler and autoscaler cannot work.
- Running application pods on the system pool.
- Letting the cluster fall out of support, then facing multiple sequential upgrades.
- Editing resources in the node resource group by hand.

## Check yourself

1. What does Microsoft manage in AKS and what remains yours?
2. Which tier gives a financially backed control plane SLA, and which adds LTS?
3. Name four defaults AKS Automatic applies that Standard does not.
4. Why should the system node pool be tainted?
5. How do surge upgrades work, and what workload configuration makes them safe?
6. What are the two upgrade channels and what does each cover?
7. When would you run multiple clusters instead of one?

<details>
<summary>Answers</summary>

1. Microsoft runs the control plane (API server, etcd, scheduler, controllers) and supplies node images; you own node pools, workloads, upgrades or their automation, add-ons and configuration.
2. Standard (and Premium) give the SLA; Premium adds Long-Term Support.
3. Any four of: node auto-provisioning; Azure CNI Overlay with Cilium; managed ingress; Entra auth with Azure RBAC; Workload Identity; deployment safeguards; image cleaner; automatic upgrades; managed monitoring; HPA, KEDA and VPA.
4. To keep application pods off nodes running critical system components, protecting cluster stability.
5. Extra nodes are added, old nodes are cordoned and drained one batch at a time, and pods move. Multiple replicas, PodDisruptionBudgets, readiness probes and graceful shutdown make it safe.
6. The cluster auto-upgrade channel (Kubernetes version) and the node OS upgrade channel (OS security patches and node images).
7. For hard isolation (security, compliance, tenants), separating prod from non-prod, limiting blast radius, multi-region deployments, or different upgrade cadences.

</details>

## References

- [AKS documentation](https://learn.microsoft.com/azure/aks/)
- [AKS Automatic](https://learn.microsoft.com/azure/aks/intro-aks-automatic)
- [Core Kubernetes concepts for AKS](https://learn.microsoft.com/azure/aks/core-aks-concepts)
- [Free, Standard and Premium tiers](https://learn.microsoft.com/azure/aks/free-standard-pricing-tiers)
- [Supported Kubernetes versions](https://learn.microsoft.com/azure/aks/supported-kubernetes-versions)
- [Long-term support](https://learn.microsoft.com/azure/aks/long-term-support)
- [Auto-upgrade and node OS upgrade channels](https://learn.microsoft.com/azure/aks/auto-upgrade-cluster)
- [AKS baseline architecture](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/baseline-aks)
- [AKS release tracker](https://releases.aks.azure.com/)
