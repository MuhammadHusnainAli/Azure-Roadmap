# AKS Scaling and Cost

> Topic: 07 · Containers and Kubernetes · Tracks: E ○ A ● C ○ · Status checked: 2026-10

## Why it matters

Kubernetes makes it easy to waste money: pods request more than they use, nodes run half empty, non-production clusters run all weekend, and GPU nodes idle. It also makes it easy to fall over under load if scaling is misconfigured. Getting scaling and cost right in AKS is mostly about accurate resource requests and letting the right autoscaler do each job.

## Key concepts

### The scaling layers

| Layer | Scales | Tool |
|---|---|---|
| Pods, horizontally | Replica count | **Horizontal Pod Autoscaler (HPA)** on CPU, memory or custom metrics |
| Pods, by events | Replica count, including to zero | **KEDA** (AKS add-on): queues, streams, Prometheus, cron, HTTP (via add-ons) |
| Pods, vertically | CPU and memory requests | **Vertical Pod Autoscaler (VPA)** |
| Nodes | Node count per pool | **Cluster autoscaler** |
| Nodes, shape and count | Chooses VM sizes and creates nodes on demand | **Node auto-provisioning (NAP)**, based on Karpenter |
| Burst | Pods on serverless capacity | Virtual nodes (ACI) |

Pod autoscalers create demand; node autoscalers satisfy it. Both depend on **resource requests**.

### Requests and limits

- **Request**: what the scheduler reserves. Drives bin packing, node scaling and cost.
- **Limit**: the maximum. Exceeding a memory limit kills the container (OOMKilled); exceeding CPU throttles it.
- Common practice: set CPU and memory requests near typical usage; set a memory limit; be cautious with CPU limits, which can throttle latency-sensitive apps.

Over-requesting is the single largest source of AKS waste. VPA in recommendation mode, or the cost analysis add-on, shows actual versus requested.

### Cluster autoscaler

Watches for pods that cannot be scheduled and adds nodes to the pool; removes nodes that are underused for a period and whose pods can move. Configure per pool (`--enable-cluster-autoscaler --min-count --max-count`) and tune with the autoscaler profile (`scale-down-delay-after-add`, `scale-down-unneeded-time`, `expander`).

### Node auto-provisioning (NAP)

With NAP, you do not size node pools. AKS looks at pending pods' requirements and provisions the best-fitting VM sizes from allowed families, and consolidates underused nodes. You constrain it with `NodePool` and `AKSNodeClass` resources (allowed SKUs, capacity type Spot or on-demand, zones, limits). AKS Automatic uses NAP by default.

| | Cluster autoscaler | Node auto-provisioning |
|---|---|---|
| Node shapes | Fixed per pool | Chosen per workload from allowed families |
| Configuration | Node pools with min and max | NodePool CRDs with constraints |
| Consolidation | Scale-down of empty or underused nodes | Active consolidation and right-sizing |
| Spot mixing | Separate Spot pools | Spot and on-demand in one NodePool definition |

### KEDA

KEDA scales deployments on external event sources and can scale to zero. Scalers include Service Bus, Event Hubs, Storage queues, Kafka, Prometheus queries, cron and many more, with Workload Identity authentication through `TriggerAuthentication`. Use it for workers and AI processing pipelines; combine with node autoscaling so nodes follow.

### Spot node pools

Spot VMs at large discounts, evicted when Azure needs capacity. Spot pools are tainted (`kubernetes.azure.com/scalesetpriority=spot:NoSchedule`); only pods that tolerate the taint run there. Good for batch, CI runners, dev environments and interruptible AI jobs with checkpoints; never for the only replica of a stateful or user-facing service.

### Cost tools

| Tool | Purpose |
|---|---|
| **AKS cost analysis add-on** (OpenCost-based) | Cost breakdown by namespace, workload and asset in Cost Management |
| Azure Advisor | Idle and underused cluster recommendations |
| Start and stop cluster | Stop non-production clusters out of hours (`az aks stop`) |
| Reservations and savings plans | Discount steady node capacity |
| Arm-based nodes (Cobalt) | Better price-performance for compatible workloads |
| Ephemeral OS disks | Cheaper and faster than managed OS disks where VM cache allows |
| Free tier control plane | For dev and test clusters |

## How it works on Azure

### HPA and KEDA examples

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: chat-api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: chat-api }
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: ingest-worker }
spec:
  scaleTargetRef: { name: ingest-worker }
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
  - type: azure-servicebus
    metadata: { namespace: sb-ai-prod, queueName: documents, messageCount: "20" }
    authenticationRef: { name: sb-workload-identity }
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata: { name: sb-workload-identity }
spec:
  podIdentity: { provider: azure-workload, identityId: "<client-id>" }
```

### A NAP NodePool allowing Spot and on-demand

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: { name: general }
spec:
  template:
    spec:
      nodeClassRef: { group: karpenter.azure.com, kind: AKSNodeClass, name: default }
      requirements:
      - { key: karpenter.azure.com/sku-family, operator: In, values: [ D, E ] }
      - { key: karpenter.sh/capacity-type, operator: In, values: [ spot, on-demand ] }
      - { key: topology.kubernetes.io/zone, operator: In, values: [ swedencentral-1, swedencentral-2, swedencentral-3 ] }
  limits: { cpu: "200" }
  disruption: { consolidationPolicy: WhenEmptyOrUnderutilized, consolidateAfter: 5m }
```

API versions of these CRDs move; check the version your cluster serves.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Cluster autoscaler or NAP | NAP for new clusters with varied workloads; cluster autoscaler where you want explicit pool control |
| HPA or KEDA | HPA for request-driven services on CPU or custom metrics; KEDA for event-driven workers and scale to zero |
| Minimum replicas | At least 2 for production services, spread across zones |
| Spot share | Use for tolerant workloads only; keep enough on-demand capacity for the service floor |
| Shared vs dedicated clusters | Sharing improves utilisation; showback per namespace keeps teams accountable |
| Non-production | Stop clusters out of hours; use the Free tier; smaller and Spot nodes |
| Reservations | Reserve the steady baseline (system pool and minimum user nodes); let autoscaling run on pay-as-you-go or Spot |

## Hands-on

```bash
RG=rg-aks-cost; AKS=aks-cost; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Cluster with autoscaler on the system pool and the cost analysis add-on (Standard tier required)
az aks create -g $RG -n $AKS --tier standard --node-count 2 \
  --enable-cluster-autoscaler --min-count 2 --max-count 5 \
  --enable-cost-analysis --generate-ssh-keys -o none
az aks get-credentials -g $RG -n $AKS --overwrite-existing

# 2. A Spot user pool
az aks nodepool add -g $RG --cluster-name $AKS -n spotpool \
  --priority Spot --eviction-policy Delete --spot-max-price -1 \
  --enable-cluster-autoscaler --min-count 0 --max-count 5 --node-vm-size Standard_D4ds_v5 -o none

# 3. Load test the HPA
kubectl create deployment php --image=registry.k8s.io/hpa-example
kubectl set resources deployment php --requests=cpu=200m --limits=memory=256Mi
kubectl expose deployment php --port 80
kubectl autoscale deployment php --cpu-percent=50 --min=1 --max=20
kubectl run load --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://php; done"
kubectl get hpa php -w            # watch replicas grow; nodes follow if pods go pending
kubectl delete pod load

# 4. Stop the cluster when you are done for the day
az aks stop -g $RG -n $AKS
az aks start -g $RG -n $AKS

az group delete -n $RG --yes --no-wait
```

After a few hours, open **Cost Management → Kubernetes** for the cluster to see cost by namespace.

## Security, cost and reliability notes

- **Cost:** right-size requests first, then autoscale, then discount (reservations, savings plans, Spot). In that order.
- **Cost:** idle GPU nodes are the most expensive waste. Use minimum 0 on GPU pools and taints so only GPU workloads land there.
- **Reliability:** scale-out is not instant: node provisioning takes minutes. Keep headroom (over-provisioning with low-priority placeholder pods) for sudden bursts.
- **Reliability:** Spot evictions arrive with about 30 seconds' notice. Handle `SIGTERM`, checkpoint, and keep critical replicas on on-demand capacity.
- **Reliability:** VPA in `Auto` mode restarts pods to apply new requests; use recommendation mode for services that cannot tolerate restarts.

## Common mistakes

- No resource requests, so autoscalers cannot work.
- Requests copied from a template and never revisited, often four times actual use.
- HPA and VPA both acting on CPU for the same deployment, fighting each other.
- Spot pools without on-demand fallback for services.
- Non-production clusters running 24/7.
- GPU nodes with a minimum count above zero for occasional jobs.
- Cluster autoscaler maximums so low that pods stay pending during peaks.

## Check yourself

1. Why are resource requests the foundation of both scaling and cost?
2. What does the cluster autoscaler react to?
3. How does node auto-provisioning differ from the cluster autoscaler?
4. When is KEDA better than HPA?
5. How do you ensure only tolerant workloads run on Spot nodes?
6. What is the order of cost optimisation steps for AKS?
7. Which tool gives cost by namespace?

<details>
<summary>Answers</summary>

1. The scheduler packs pods by request, node autoscalers add nodes based on unschedulable requests, and you pay for the nodes. Inflated requests mean more nodes and higher cost.
2. Pods that are pending because no node has enough requested capacity, and underused nodes that could be removed.
3. NAP chooses VM sizes per workload from allowed families and consolidates actively, instead of scaling fixed-shape node pools within min and max counts.
4. For event-driven workloads scaled on external metrics such as queue length, and when you need scale to zero.
5. Spot pools carry a taint; only pods with a matching toleration (and ideally node affinity) are scheduled there.
6. Right-size requests, autoscale pods and nodes, stop or shrink idle environments, then apply reservations, savings plans and Spot.
7. The AKS cost analysis add-on in Cost Management.

</details>

## References

- [Scaling options for AKS](https://learn.microsoft.com/azure/aks/concepts-scale)
- [Cluster autoscaler](https://learn.microsoft.com/azure/aks/cluster-autoscaler)
- [Node auto-provisioning](https://learn.microsoft.com/azure/aks/node-autoprovision)
- [KEDA add-on](https://learn.microsoft.com/azure/aks/keda-about)
- [Vertical Pod Autoscaler](https://learn.microsoft.com/azure/aks/vertical-pod-autoscaler)
- [Spot node pools](https://learn.microsoft.com/azure/aks/spot-node-pool)
- [AKS cost analysis](https://learn.microsoft.com/azure/aks/cost-analysis)
- [Cost optimisation best practices for AKS](https://learn.microsoft.com/azure/aks/best-practices-cost)
- [Start and stop a cluster](https://learn.microsoft.com/azure/aks/start-stop-cluster)
