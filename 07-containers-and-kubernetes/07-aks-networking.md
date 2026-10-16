# AKS Networking

> Topic: 07 · Containers and Kubernetes · Tracks: E ○ A ● C – · Status checked: 2026-10

## Why it matters

Networking is where most AKS designs go wrong: IP exhaustion, blocked egress, broken private DNS, or an ingress layer nobody can upgrade. Choices made at cluster creation, especially the network plugin, are hard or impossible to change later. An architect must choose the IP model, ingress, egress and policy approach deliberately, and fit them into the hub-and-spoke design from [topic 05](../05-networking/README.md).

## Key concepts

### Network plugin models

| Model | Pod IPs | IP usage in VNet | Status |
|---|---|---|---|
| **Azure CNI Overlay** | From a private overlay CIDR, not the VNet | Only nodes consume VNet IPs | **Default and recommended** |
| **Azure CNI (VNet, dynamic pod IP allocation)** | From a dedicated pod subnet in the VNet | Pods consume VNet IPs | When pods must be directly routable from the VNet |
| **Azure CNI (node subnet, static)** | From the node subnet, pre-allocated per node | Heavy | Legacy pattern; avoid for new clusters |
| **Kubenet** | Overlay with route tables | Nodes only | **Retiring (March 2028)**; migrate to CNI Overlay |
| **Bring your own CNI** | Whatever you install | Varies | Specialist cases only |

With Overlay, pods reach VNet resources with SNAT to the node IP; resources in the VNet cannot initiate connections directly to pod IPs (they go through Services and load balancers, which is normal practice).

### Data plane: Cilium

**Azure CNI powered by Cilium** uses eBPF instead of iptables for routing, Services and network policy. It scales better, enforces policy efficiently and enables advanced observability. Choose it at creation (`--network-dataplane cilium`). AKS Automatic uses it by default.

**Advanced Container Networking Services** (ACNS) adds, on top of Cilium: network flow observability (Hubble-based flow logs and metrics), FQDN-based network policies, Layer 7 policies, and container network security features. It is a paid add-on.

### Network policy

By default every pod can talk to every pod. Network policies restrict that.

| Engine | Notes |
|---|---|
| **Cilium** | Recommended with Cilium data plane; supports FQDN and L7 policy with ACNS |
| Azure Network Policy Manager | Older; being phased out in favour of Cilium |
| Calico | Supported option, open-source and enterprise |

Start with a default-deny policy per namespace, then allow specific flows.

### Services and load balancers

| Service type | Exposure |
|---|---|
| `ClusterIP` | Internal to the cluster |
| `LoadBalancer` | Azure Standard Load Balancer, public or internal (`service.beta.kubernetes.io/azure-load-balancer-internal: "true"`) |
| `NodePort` | Rarely useful on AKS |
| Headless | Direct pod DNS for stateful sets |

The Basic Load Balancer is retired; AKS uses Standard Load Balancer.

### Ingress and Gateway API

| Option | What it is | Notes |
|---|---|---|
| **Application routing add-on** | Managed ingress controller with Azure DNS and Key Vault certificate integration | Historically NGINX-based. The upstream ingress-nginx project was retired in 2026; AKS is moving application routing to Gateway API implementations. Check the current migration guidance |
| **Application Gateway for Containers** | Azure-managed L7 load balancer outside the cluster, supports **Gateway API** and Ingress, WAF integration | Recommended Azure-native option for production HTTP ingress |
| **Istio ingress gateway** (with the Istio add-on) | Mesh-integrated ingress, Gateway API support | When you already use the mesh |
| Application Gateway Ingress Controller (AGIC) | Older integration with Application Gateway v2 | Prefer Application Gateway for Containers for new work |
| Self-managed (Traefik, Envoy Gateway, Contour, others) | You run and patch it | Full control, full responsibility |

**Gateway API** is the successor to Ingress: role-oriented resources (`GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`), richer routing (header matching, traffic splitting), and portability across implementations. New designs should target Gateway API.

### Service mesh

The **Istio-based service mesh add-on** provides managed Istio: mutual TLS between services, traffic management (retries, timeouts, canary splits), and telemetry. Use a mesh when you need zero-trust mTLS everywhere or fine traffic control across many services; it adds latency, memory and operational complexity. Cilium also offers mTLS and some mesh features without sidecars.

### Egress

| Outbound type | Detail |
|---|---|
| `loadBalancer` (default) | SNAT through the cluster load balancer's public IP |
| `managedNATGateway` / `userAssignedNATGateway` | NAT gateway, better SNAT port scale |
| `userDefinedRouting` | Route 0.0.0.0/0 to Azure Firewall or an NVA in the hub; required for inspected egress |
| `none` | No outbound configured (for fully isolated clusters with private dependencies) |

With a firewall, allow the **AKS required outbound** FQDNs and ports (Microsoft Container Registry, management endpoints, node image updates, monitoring endpoints). Missing rules cause failed node provisioning and upgrades. Azure Firewall has an `AzureKubernetesService` FQDN tag for this.

### API server access

| Option | Detail |
|---|---|
| Public with authorised IP ranges | API server public, restricted to listed CIDRs |
| **Private cluster** | API server reachable only through a private endpoint; needs private DNS and a network path for CI and admins |
| **API server VNet integration** | API server projected into a delegated subnet in your VNet; private without a private endpoint, and can be toggled public or private |

Use Azure Bastion, a jump box, self-hosted CI agents or `az aks command invoke` to operate private clusters.

## How it works on Azure

### Reference design in a hub-and-spoke

```text
Hub VNet: Azure Firewall, Bastion, DNS resolver, VPN/ExpressRoute
   ▲ peering
Spoke VNet 10.20.0.0/16
 ├─ snet-aks-nodes   10.20.0.0/22      (node IPs; Overlay pods use 192.168.0.0/16)
 ├─ snet-aks-api     10.20.4.0/28      (API server VNet integration)
 ├─ snet-agc         10.20.5.0/24      (Application Gateway for Containers)
 ├─ snet-pe          10.20.6.0/24      (private endpoints: ACR, Key Vault, Foundry, Storage)
 UDR 0.0.0.0/0 → hub firewall (outboundType userDefinedRouting)
 Private DNS zones linked to the hub resolver
Ingress: Front Door (WAF) → Application Gateway for Containers → HTTPRoutes → Services
```

### IP planning with Overlay

- Node subnet: one IP per node plus surge during upgrades; size for maximum node count across all pools plus headroom.
- Pod CIDR: a private range not used anywhere else you need to reach (default `10.244.0.0/16`); overlays can be reused across clusters.
- Service CIDR: another non-overlapping range for ClusterIPs (`--service-cidr`, `--dns-service-ip`).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Plugin | Azure CNI Overlay with Cilium data plane by default |
| Pods directly routable from VNet | Only if required; then Azure CNI with dynamic pod IP allocation and generous subnets |
| Ingress | Application Gateway for Containers for Azure-native, WAF-capable ingress using Gateway API; Istio gateway if using the mesh |
| Network policy | Cilium, default deny per namespace |
| Mesh | Only when mTLS and traffic control across many services justify the overhead |
| Egress | UDR to the hub firewall for regulated environments; NAT gateway otherwise |
| API server | Private (VNet integration or private cluster) for production, with a planned admin path |

## Hands-on

```bash
RG=rg-aks-net; LOC=swedencentral; AKS=aks-net
az group create -n $RG -l $LOC -o none

# 1. Overlay + Cilium cluster
az aks create -g $RG -n $AKS --node-count 2 \
  --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium \
  --pod-cidr 192.168.0.0/16 --generate-ssh-keys -o none
az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubectl get nodes -o wide                         # node IPs from the VNet
kubectl get pods -A -o wide | head                # pod IPs from 192.168.x.x

# 2. Two apps and a default-deny policy
kubectl create ns shop
kubectl run web --image=nginx -n shop --labels app=web --port 80
kubectl expose pod web -n shop --port 80
kubectl run client --image=busybox -n shop --labels app=client -- sleep 3600
kubectl exec -n shop client -- wget -qO- --timeout=3 web | head -2      # works

cat <<'EOF' | kubectl apply -n shop -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-ingress }
spec: { podSelector: {}, policyTypes: [ Ingress ] }
EOF
kubectl exec -n shop client -- wget -qO- --timeout=3 web || echo "blocked"

cat <<'EOF' | kubectl apply -n shop -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-client-to-web }
spec:
  podSelector: { matchLabels: { app: web } }
  ingress: [ { from: [ { podSelector: { matchLabels: { app: client } } } ], ports: [ { port: 80 } ] } ]
EOF
kubectl exec -n shop client -- wget -qO- --timeout=3 web | head -2      # works again

# 3. Internal load balancer service
kubectl expose pod web -n shop --name web-ilb --type LoadBalancer --port 80 \
  --overrides='{"metadata":{"annotations":{"service.beta.kubernetes.io/azure-load-balancer-internal":"true"}}}'
kubectl get svc web-ilb -n shop -w

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** default-deny network policies, private API server, egress through a firewall, WAF in front of public ingress, private endpoints for all PaaS dependencies.
- **Security:** with a firewall, prefer FQDN rules for the AKS required endpoints over broad IP ranges.
- **Cost:** Azure Firewall and Application Gateway for Containers are significant fixed costs; share hubs across clusters. ACNS is billed per node.
- **Reliability:** undersized subnets block scale-out and upgrades (surge nodes need IPs). Plan for maximum scale plus surge.
- **Reliability:** SNAT port exhaustion on busy egress; use NAT gateway or a firewall with enough public IPs.

## Common mistakes

- Creating new clusters with kubenet or node-subnet CNI, then running out of IPs.
- Overlapping pod or service CIDRs with on-premises ranges that pods need to reach.
- Private cluster with no plan for how CI/CD reaches the API server.
- Forced tunnelling to a firewall without the AKS required outbound rules, breaking node provisioning.
- No network policies: one compromised pod can reach everything.
- Building new ingress on the retired community ingress-nginx project.
- Adding a service mesh "because microservices", then struggling with its upgrades.

## Check yourself

1. Why is Azure CNI Overlay the default, and what is its main limitation?
2. What does Cilium change compared with iptables-based networking?
3. What is the Gateway API and why prefer it over Ingress for new designs?
4. Which outbound type sends egress through Azure Firewall?
5. What happens if required outbound FQDNs are blocked?
6. What are the options for making the API server private?
7. When is a service mesh justified?

<details>
<summary>Answers</summary>

1. Pods use an overlay CIDR so only nodes consume VNet IPs, avoiding exhaustion. Pods are not directly reachable from outside the cluster by pod IP.
2. It uses eBPF for routing, services and policy, which scales better, is faster to update, and enables L7, FQDN policy and deep flow observability.
3. A role-oriented, expressive, portable successor to Ingress with richer routing such as traffic splitting and header matching, supported by multiple implementations including Application Gateway for Containers and Istio.
4. `userDefinedRouting`, with a UDR sending 0.0.0.0/0 to the firewall.
5. Nodes fail to provision or upgrade, images fail to pull, and the cluster can become unhealthy.
6. Private cluster (private endpoint) or API server VNet integration; or authorised IP ranges on a public endpoint as a lesser control.
7. When you need mTLS between many services, fine-grained traffic management or consistent service telemetry, and have the capacity to operate it.

</details>

## References

- [AKS networking concepts](https://learn.microsoft.com/azure/aks/concepts-network)
- [Azure CNI Overlay](https://learn.microsoft.com/azure/aks/azure-cni-overlay)
- [Azure CNI powered by Cilium](https://learn.microsoft.com/azure/aks/azure-cni-powered-by-cilium)
- [Advanced Container Networking Services](https://learn.microsoft.com/azure/aks/advanced-container-networking-services-overview)
- [Application Gateway for Containers](https://learn.microsoft.com/azure/application-gateway/for-containers/overview)
- [Application routing add-on](https://learn.microsoft.com/azure/aks/app-routing)
- [Istio-based service mesh add-on](https://learn.microsoft.com/azure/aks/istio-about)
- [Control egress traffic with Azure Firewall](https://learn.microsoft.com/azure/aks/limit-egress-traffic)
- [Private clusters](https://learn.microsoft.com/azure/aks/private-clusters) and [API server VNet integration](https://learn.microsoft.com/azure/aks/api-server-vnet-integration)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
