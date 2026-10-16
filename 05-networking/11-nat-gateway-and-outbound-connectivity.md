# NAT Gateway and Outbound Connectivity

> Topic: 05 · Networking · Tracks: E ● A ● C – · Status checked: 2026-10

## Why it matters

Outbound connectivity used to happen by accident in Azure. It no longer does: new virtual networks are private by default, so a workload that needs to reach the internet needs a designed path. Getting this wrong shows up as a deployment that cannot pull a container image, or, later and worse, as intermittent connection failures under load from SNAT port exhaustion. AI applications make many outbound calls and hit this more than most.

## Key concepts

### The end of default outbound access

Previously, a VM with no public IP and no explicit configuration still reached the internet through an implicit, Azure-owned IP that could change without notice.

**Default outbound access was retired for new virtual networks on 31 March 2026.** With API versions released after that date, new subnets are created as **private subnets** (`defaultOutboundAccess: false`). Existing virtual networks that depended on the old behaviour keep working, but you should not build on it.

In a private subnet, a resource has **no path to the internet** until you provide one of the explicit methods below.

### Outbound methods

| Method | How it works | Use when |
|---|---|---|
| **NAT Gateway** | A managed service attached to a subnet. All outbound flows are source-NATed to its public IPs | The default for subnets that need direct outbound access |
| **Azure Firewall or NVA** | A UDR sends `0.0.0.0/0` to the firewall, which SNATs to its public IP | You need filtering, inspection and logging of egress. Standard in landing zones |
| **Standard Load Balancer outbound rules** | Backend pool members share the load balancer's public IP for SNAT | You already have a public load balancer and modest outbound needs |
| **Instance-level public IP** | A public IP on the VM's NIC | A specific VM needs its own address. Avoid for general use |

If more than one applies to a resource, precedence for internet-bound traffic is: a **UDR** to a virtual appliance or gateway first; otherwise **NAT Gateway**; otherwise an instance public IP; otherwise load balancer outbound rules.

### SNAT and port exhaustion

Source network address translation rewrites a private source IP and port to a public IP and port. Each public IP has 64,512 usable ports, and each **concurrent connection to the same destination IP and port** consumes one.

When a source runs out of ports, new connections fail or stall. Symptoms: intermittent timeouts under load, errors such as *connection timed out* or *cannot assign requested address*, problems that vanish at low traffic.

Causes are nearly always application behaviour:

- Opening a new connection per request instead of reusing connections.
- No connection pooling, or pools created per request.
- Many instances behind too few public IPs.
- Long idle timeouts holding ports.

### NAT Gateway

A fully managed, resilient outbound NAT service.

| Property | Detail |
|---|---|
| **Direction** | Outbound only. Return traffic for an outbound flow is allowed; unsolicited inbound is not |
| **Scope** | Associated with one or more **subnets** in a single VNet |
| **Addresses** | Up to 16 public IPs, or a public IP prefix |
| **Ports** | 64,512 per IP, allocated **on demand** across all resources in the subnet, so over 1 million with 16 IPs |
| **Protocols** | TCP and UDP |
| **Idle timeout** | TCP default 4 minutes, configurable up to 120 |
| **Throughput** | Tens of Gbps |
| **Benefit over load balancer SNAT** | Dynamic port allocation instead of fixed per-instance allocation, so exhaustion is far less likely |
| **Predictable egress IP** | Destinations can allow-list your NAT gateway's addresses |

**SKUs**

| SKU | Availability zones |
|---|---|
| **Standard** | **Zonal**: deployed in one zone (or no zone). A zone failure affects subnets using it. For zone resilience with Standard you align one NAT gateway per zone with zonal subnets |
| **StandardV2** | **Zone-redundant**: spans zones automatically. Also adds higher throughput and features such as flow logs and IPv6 support |

Use StandardV2 for new deployments where it is available in your region; confirm current regional availability and any feature differences.

What NAT Gateway does **not** do: filter traffic, inspect it, or log destinations by name. It is plumbing. For control, use a firewall.

### NAT Gateway with other services

| Service | Outbound behaviour | With NAT Gateway |
|---|---|---|
| **Virtual machines / scale sets** | Subnet's outbound method | Attach to the subnet |
| **App Service / Functions** | Shared platform IPs by default | With **VNet integration** and *route all* enabled, outbound goes through the integration subnet's NAT gateway, giving a dedicated egress IP and more SNAT ports |
| **Container Apps** | Environment's outbound IPs | In a custom VNet with a workload profiles environment, attach a NAT gateway to the infrastructure subnet, or route through a firewall with a UDR |
| **AKS** | `outboundType`: `loadBalancer`, `managedNATGateway`, `userAssignedNATGateway`, `userDefinedRouting`, or `none` for isolated clusters | Choose a NAT gateway type for large clusters or many outbound connections |
| **Azure Firewall** | Its own public IPs | Attach a NAT gateway to `AzureFirewallSubnet` to scale SNAT ports well beyond the firewall's own |
| **Foundry Agent Service (network-isolated)** | Through the delegated subnet's routing | Egress follows the subnet's UDR or NAT gateway |

### Things that depended on default outbound

Moving to private subnets surfaces hidden dependencies:

| Dependency | What to do |
|---|---|
| Windows activation (KMS) and Windows Update | Provide an explicit outbound path, or use the documented service tags and FQDNs through the firewall |
| VM extensions downloading packages | Allow the required storage and package endpoints, or use private endpoints |
| OS package repositories | Allow through the firewall, or mirror internally |
| Agents phoning home (monitoring, security) | Private Link scopes or firewall rules |
| `168.63.129.16` and the instance metadata service | Unaffected: these are platform addresses inside the VNet |

## How it works on Azure

### Choosing an egress design

```text
Does the organisation require filtering or inspection of outbound traffic?
 ├─ Yes ─► UDR to Azure Firewall (or NVA) in the hub.
 │         High connection volume? Add a NAT gateway to the firewall subnet.
 └─ No  ─► Does the workload need internet access at all?
            ├─ No  ─► Private subnet, private endpoints only. No egress path. (Most secure.)
            └─ Yes ─► NAT gateway on the subnet. Zone-redundant (StandardV2) where available.
```

Many AI workloads need **less** egress than expected. If the model, search, storage, database, registry and Key Vault are all reached by private endpoint, and telemetry goes through a private link scope, the application may need no internet path at all. Agent tools that browse or call external APIs are the usual exception; give those a controlled path.

### Avoiding SNAT exhaustion in applications

The durable fix is in code, not in more IPs:

- **Reuse clients.** Create HTTP and SDK clients once per process (see topic 04, file 09). This single change resolves most cases.
- **Use connection pooling** and keep-alive.
- **Prefer private endpoints** for Azure services. Traffic to a private endpoint is not SNATed and consumes no ports.
- **Use HTTP/2** where supported, multiplexing many requests over one connection.
- **Retry with backoff**, not immediately, or retries amplify the problem.
- **Size the egress:** add IPs to the NAT gateway before a known scale-up.

Calls from an app to a model endpoint over the public internet at high concurrency are a classic exhaustion scenario. A private endpoint removes it entirely.

### Monitoring

| Signal | Where |
|---|---|
| **SNAT connection count**, **dropped packets**, **datapath availability** | NAT Gateway metrics |
| **SNAT port utilisation** | Azure Firewall and Load Balancer metrics |
| **Used and allocated SNAT ports** per backend instance | Load Balancer metrics |
| Connection failures, dependency timeouts | Application Insights |
| Flow logs | NAT Gateway StandardV2 flow logs; VNet flow logs |

Alert on dropped packets and high SNAT utilisation before users notice.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| NAT Gateway or firewall | NAT gateway for simple, scalable outbound. Firewall when you need control. They also combine |
| Per-spoke NAT gateway or central egress | Central firewall egress in a landing zone. Per-spoke NAT gateways for standalone workloads or where direct egress is acceptable |
| How many public IPs | Start with one or two. Scale on measured SNAT usage. A prefix keeps addresses contiguous for partner allow-lists |
| Zonal or zone-redundant | Zone-redundant (StandardV2) where available; otherwise a zonal NAT gateway per zone aligned with zonal subnets |
| No egress at all | Best where achievable. Every allowed destination is something to justify |
| Instance public IPs | Avoid. They also expose an inbound surface unless NSGs block it |

**Cost.** NAT Gateway bills an hourly resource charge plus a per-GB data processing charge. Heavy outbound data, such as large downloads or bulk transfers, makes the per-GB part noticeable. Keep Azure-to-Azure traffic on private endpoints and it never touches the NAT gateway.

## Hands-on

```bash
RG=rg-nat-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. A VNet with a private subnet (no implicit outbound)
az network vnet create -g $RG -n vnet-nat --address-prefixes 10.40.0.0/16 -o none
az network vnet subnet create -g $RG --vnet-name vnet-nat -n snet-app --address-prefixes 10.40.1.0/24 \
  --default-outbound false -o none
az network vnet subnet show -g $RG --vnet-name vnet-nat -n snet-app --query defaultOutboundAccess

# 2. A small VM without a public IP
az vm create -g $RG -n vm-nat --image Ubuntu2404 --size Standard_B2s \
  --vnet-name vnet-nat --subnet snet-app --public-ip-address "" --nsg "" \
  --admin-username azureuser --generate-ssh-keys -o none

# 3. Prove there is no outbound path
az vm run-command invoke -g $RG -n vm-nat --command-id RunShellScript \
  --scripts "curl -s -m 8 https://ifconfig.me || echo NO-OUTBOUND" --query "value[0].message" -o tsv

# 4. Add a NAT gateway
az network public-ip create -g $RG -n pip-nat --sku Standard --zone 1 -o none
az network nat gateway create -g $RG -n ng-lab --public-ip-addresses pip-nat --idle-timeout 4 --zone 1 -o none
az network vnet subnet update -g $RG --vnet-name vnet-nat -n snet-app --nat-gateway ng-lab -o none

# 5. Outbound now works, from the NAT gateway's address
az network public-ip show -g $RG -n pip-nat --query ipAddress -o tsv
az vm run-command invoke -g $RG -n vm-nat --command-id RunShellScript \
  --scripts "curl -s -m 8 https://ifconfig.me" --query "value[0].message" -o tsv

# 6. Metrics
NG_ID=$(az network nat gateway show -g $RG -n ng-lab --query id -o tsv)
az monitor metrics list --resource $NG_ID --metric SNATConnectionCount --interval PT1M -o table | tail -5

az group delete -n $RG --yes --no-wait
```

The address returned in step 5 should match the NAT gateway's public IP. Try `--sku StandardV2` for the NAT gateway and its public IP if your region supports it, and note the zone behaviour.

Paper exercise: an API on App Service makes 2,000 concurrent calls to a partner API and fails intermittently with timeouts. List three changes, in priority order.

<details>
<summary>Suggested answer</summary>

1. Reuse a single HTTP client with connection pooling and keep-alive (and HTTP/2 if the partner supports it).
2. Enable VNet integration with all outbound traffic routed, and attach a NAT gateway to the integration subnet for dynamically allocated ports.
3. Add public IPs to the NAT gateway if SNAT metrics still show pressure, and add retry with backoff.

</details>

## Security, cost and reliability notes

- **Security:** a NAT gateway allows outbound to **anywhere**. It gives no control over destinations. Pair with NSG outbound rules at minimum, or use a firewall.
- **Security:** a fixed egress IP is useful for partner allow-lists but is weak authentication by itself. Combine with real credentials.
- **Security:** the most secure egress is none. Default to private subnets and add paths by exception.
- **Cost:** hourly plus per-GB. Private endpoints for Azure services both reduce NAT charges and improve security.
- **Reliability:** a zonal NAT gateway is a zonal dependency. Design for zone redundancy explicitly.
- **Reliability:** idle timeouts drop long-lived silent connections. Use TCP keep-alives for long-running streams.

## Common mistakes

- Assuming a new VM can reach the internet.
- Discovering hidden dependencies on default outbound (updates, extensions, activation) in production.
- One HTTP client per request.
- Reaching Azure PaaS services over public endpoints at high concurrency instead of private endpoints.
- A single zonal NAT gateway serving a zone-redundant application.
- Treating NAT Gateway as a security control.
- Adding a NAT gateway to a subnet that already has a `0.0.0.0/0` UDR to a firewall and expecting the NAT gateway to be used.
- Not monitoring SNAT metrics.

## Check yourself

1. What changed about outbound access for new virtual networks, and when?
2. List the four explicit outbound methods.
3. If a subnet has both a NAT gateway and a UDR for `0.0.0.0/0` to a firewall, which is used?
4. What is SNAT port exhaustion, and what is its most common cause?
5. Why is NAT Gateway less prone to exhaustion than load balancer outbound rules?
6. What is the zone behaviour of Standard versus StandardV2 NAT Gateway?
7. Why does using private endpoints help with SNAT?
8. Does a NAT gateway filter outbound traffic?

<details>
<summary>Answers</summary>

1. Default outbound access was retired for new virtual networks on 31 March 2026; new subnets are private by default and need an explicit outbound method.
2. NAT Gateway, a firewall or NVA through a UDR, Standard Load Balancer outbound rules, an instance-level public IP.
3. The UDR to the firewall.
4. Running out of source ports for outbound connections to the same destination, usually caused by applications creating new connections per request instead of reusing them.
5. It allocates ports dynamically on demand across the whole subnet from a large pool, instead of pre-allocating a fixed number per instance.
6. Standard is zonal (one zone); StandardV2 is zone-redundant.
7. Traffic to a private endpoint stays on private addresses and is not source-NATed, so it consumes no SNAT ports.
8. No. It only translates addresses.

</details>

## References

- [Default outbound access in Azure](https://learn.microsoft.com/azure/virtual-network/ip-services/default-outbound-access)
- [What is Azure NAT Gateway?](https://learn.microsoft.com/azure/nat-gateway/nat-overview)
- [Azure NAT Gateway resource and SKUs](https://learn.microsoft.com/azure/nat-gateway/nat-gateway-resource)
- [Design virtual networks with NAT gateway](https://learn.microsoft.com/azure/nat-gateway/nat-gateway-design)
- [Source Network Address Translation (SNAT) with Azure NAT Gateway](https://learn.microsoft.com/azure/nat-gateway/nat-gateway-snat)
- [Use SNAT for outbound connections (Load Balancer)](https://learn.microsoft.com/azure/load-balancer/load-balancer-outbound-connections)
- [Troubleshoot intermittent outbound connection errors in App Service](https://learn.microsoft.com/azure/app-service/troubleshoot-intermittent-outbound-connection-errors)
- [Azure NAT Gateway integration with App Service](https://learn.microsoft.com/azure/app-service/networking/nat-gateway-integration)
- [Outbound network and FQDN rules for AKS](https://learn.microsoft.com/azure/aks/outbound-rules-control-egress)
