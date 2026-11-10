# Network Watcher and Troubleshooting

## Why it matters

"It can't connect" is the most common infrastructure problem you will be handed, and in a private Azure environment there are five or six layers that could be responsible. Engineers who troubleshoot by guessing lose hours; those with a method and the right tools find the cause in minutes. This file gives you both. It is one of the most practically useful files in the networking topic.

## Key concepts

### The layers a connection passes through

For a client to reach a service, every one of these must succeed, in roughly this order:

```text
1. DNS            the name resolves to the intended IP
2. Routing        packets have a path there (and back)
3. Filtering      NSGs, security admin rules, firewalls and service firewalls allow it
4. Listener       something is listening on that IP and port
5. TLS            certificates and host names match
6. Authentication the caller presents a valid token for the right audience
7. Authorisation  the caller has the role, and policy allows it
8. Application    the service itself responds correctly
```

Work **top to bottom**. Most "network" problems are DNS (1) or filtering (3). Many reported network problems are actually 6 or 7.

### Reading the symptom

| Symptom | Usually means |
|---|---|
| **Name does not resolve** (NXDOMAIN) | DNS: missing record, zone not linked, wrong resolver |
| **Resolves to a public IP** from inside a private network | DNS: private zone not linked to the resolving VNet, or client bypassing Azure DNS |
| **Connection timed out** | Packets dropped silently: NSG deny, firewall deny, no route, asymmetric routing, service not reachable from that network |
| **Connection refused** (TCP reset) | Reached the host, nothing listening on the port; or a device actively rejecting |
| **TLS / certificate error** | Host name mismatch (connecting by IP, wrong host header), untrusted CA, TLS inspection without the CA installed |
| **HTTP 401** | Authentication: missing or invalid token, wrong audience, expired |
| **HTTP 403** | Authorisation or **network rule**: missing RBAC role, or the service firewall rejects the source (public access disabled and caller not on a private path). Read the error body; it usually says which |
| **HTTP 404** | Wrong endpoint, deployment name, path or API version |
| **HTTP 429** | Throttling: quota or rate limit, not a network fault |
| **HTTP 502 / 504** from a gateway | The proxy could not reach or did not get a timely answer from its backend |
| **Intermittent failures under load** | SNAT port exhaustion, scaling limits, timeouts |
| **Works from one place, not another** | Compare DNS answers and network paths between the two |

Timeouts versus refusals is the single most useful distinction: a **timeout** means something dropped the packets; a **refusal** means you got there.

### Azure Network Watcher

A regional service with a set of diagnostic and monitoring tools. It is enabled automatically per region when a VNet exists.

**Diagnostic tools**

| Tool | Question it answers |
|---|---|
| **IP flow verify** | Would a packet with this 5-tuple to or from this VM be allowed or denied, and by **which NSG rule**? |
| **NSG diagnostics** | For traffic between a source and destination, list every NSG and rule evaluated and the outcome |
| **Next hop** | For a packet from this VM to this IP, what is the next hop type and IP, and which route table supplied it? |
| **Effective security rules** | The combined NSG rules applying to a NIC |
| **Effective routes** | The combined route table for a NIC |
| **Connection troubleshoot** | Test a connection from a VM, Bastion, Application Gateway or scale set to a destination (VM, FQDN, IP): reachability, latency, hop-by-hop path, and which NSG rule or route blocks it |
| **Packet capture** | Capture traffic on a VM (through the Network Watcher extension) to a storage account or file, with filters |
| **VPN troubleshoot** | Diagnose a VPN gateway or connection |
| **Topology** | A visual map of resources and their relationships |

**Monitoring tools**

| Tool | Use |
|---|---|
| **Connection Monitor** | Continuous synthetic tests between sources (Azure VMs, scale sets, Arc-enabled on-premises machines) and destinations (VMs, URLs, IPs), measuring reachability, latency and loss, with alerts and path visualisation |
| **VNet flow logs** | Records of IP flows through a virtual network: allowed and denied, by which rule. Replaces NSG flow logs, which retire on 30 September 2027 |
| **Traffic Analytics** | Processes flow logs into Log Analytics for querying and visual insight |

**Related**

- **Azure Virtual Network Manager verifier**: static reachability analysis across your network configuration without sending traffic.
- **Azure Monitor Network Insights**: a health and metrics overview of all network resources.
- **Resource health / Service Health**: is the platform itself having a problem?

### Command-line essentials

From a machine inside the network:

```bash
# 1. DNS
nslookup <host>                    # which IP, via which CNAMEs?
dig +short <host>
cat /etc/resolv.conf               # which resolver? (ipconfig /all on Windows)

# 2-4. Reachability of a TCP port
nc -zv <host> 443                  # Linux
curl -v telnet://<host>:443
Test-NetConnection <host> -Port 443            # PowerShell
tcpping <host> 443                 # or psping on Windows

# 5-8. TLS and HTTP in one go
curl -sv https://<host>/ -o /dev/null          # shows resolved IP, TLS handshake, certificate, status
openssl s_client -connect <host>:443 -servername <host> </dev/null | head -20

# Outbound IP as seen by the internet
curl -s https://ifconfig.me

# Route and interface information on the VM
ip route ; ip addr                 # route print ; ipconfig on Windows
```

`ping` (ICMP) is a poor test in Azure: many services and load balancers do not answer it, and NSGs often block it. Test the actual TCP port.

`curl -v` is the best single command: it prints the IP it connected to, the TLS result and the HTTP status, which locates the failing layer immediately.

From Azure, without being on the machine:

```bash
# NSG decision for a specific flow
az network watcher test-ip-flow --vm <vm> -g <rg> --direction Outbound --protocol TCP \
  --local 10.20.1.4:50000 --remote 10.20.3.5:443

# Where would the packet go?
az network watcher show-next-hop --vm <vm> -g <rg> --source-ip 10.20.1.4 --dest-ip 10.20.3.5

# End-to-end test with hop detail
az network watcher test-connectivity -g <rg> --source-resource <vm> \
  --dest-address <fqdn-or-ip> --dest-port 443

# Merged rules and routes on a NIC
az network nic list-effective-nsg -g <rg> -n <nic>
az network nic show-effective-route-table -g <rg> -n <nic> -o table

# Run a test on a VM with no network access to it
az vm run-command invoke -g <rg> -n <vm> --command-id RunShellScript \
  --scripts "nslookup <host>; curl -sv https://<host>/ -o /dev/null 2>&1 | head -30"
```

For PaaS hosts without a shell you can reach: App Service has a console (Kudu) with `nameresolver` and `tcpping`; Container Apps has `az containerapp exec` or a debug console; AKS has `kubectl debug` and ephemeral containers.

## How it works on Azure

### A method that works

**Step 0: define the flow precisely.** Source (which resource, which subnet, which IP), destination (name, IP, port), protocol, and what exactly fails with what error. Half of all troubleshooting is discovering the flow is not what was assumed.

**Step 1: DNS.** From the **source**, resolve the destination name.

- Private resource expected: is the answer a private IP?
- If not: check the private DNS zone exists, has the record, and is linked to the VNet doing the resolving (or the hub, if resolution is centralised); check the client's DNS server.

**Step 2: Is it reaching the right place?** `curl -v` or a TCP test from the source.

- **Timeout:** go to steps 3 and 4.
- **Refused:** the target is not listening, or wrong port.
- **TLS error:** host name or certificate; are you connecting by IP?
- **HTTP status:** the network is fine. Go to step 6.

**Step 3: Routing.** *Next hop* from the source to the destination IP.

- Is the next hop what you expect (VNet, peering, firewall, gateway)?
- If it is a firewall: is there a rule, and what do firewall logs say?
- Check the **return path**: next hop from the destination back to the source. Asymmetry is the classic cause of timeouts through firewalls.

**Step 4: Filtering.** *IP flow verify* or *NSG diagnostics* at the source (outbound) and destination (inbound).

- Which rule decides? Remember subnet and NIC NSGs both apply, and security admin rules come first.
- For private endpoints: are network policies enabled, and does the PE subnet NSG allow the source?
- Azure Firewall: query `AZFWNetworkRule` and `AZFWApplicationRule` for the source IP.
- Destination service firewall: is public access disabled? Is the caller arriving through a private endpoint? Is a network security perimeter enforcing?

**Step 5: The platform.** Resource health of the target, the gateway, the firewall. Service Health for the region.

**Step 6: Identity.** For 401 and 403, decode the token or read the error:

- **401:** token audience wrong (for example a management token sent to a data-plane endpoint), expired, or absent.
- **403:** read the message. *"...does not have permission..."* / *"AuthorizationPermissionMismatch"* is RBAC: check the role, the **scope**, the principal ID, and allow a few minutes for propagation. *"Public access is disabled"* / *"...not allowed by firewall rules"* / *"Traffic is not from an approved private endpoint"* is the network path.

**Step 7: Application.** Logs, traces, and the service's own diagnostics.

### Worked example: a container app cannot call a private Foundry endpoint

| Step | Check | Finding |
|---|---|---|
| 0 | App in `snet-apps` calls `https://aif-hr.openai.azure.com`, gets a 403 | – |
| 1 | `nslookup` from the container: returns `20.x.x.x` (public) | **DNS is wrong** |
| 1a | Zone `privatelink.openai.azure.com` exists with an A record | Record is fine |
| 1b | Zone links: linked to the hub only. Spoke VNet uses Azure-provided DNS | The spoke resolves through Azure DNS in its own VNet, which has no link |
| Fix | Set the spoke's DNS servers to the hub resolver or firewall proxy (central design), or link the zone to the spoke | `nslookup` now returns `10.20.3.6` |
| 2 | `curl -v` returns 200 with a valid token | Resolved |

The 403 looked like a permissions problem. It was the service rejecting a request that arrived on the public path because DNS sent it there.

### Common failure patterns

| Pattern | Cause | Fix |
|---|---|---|
| Private endpoint works from the VNet, not from on-premises | On-premises DNS not forwarding the public zone name to Azure | Conditional forwarder to the resolver's inbound endpoint |
| Works by IP, fails by name with a certificate error | Connecting by IP | Always use the FQDN; fix DNS |
| Some Foundry calls work, others time out | One of the three `privatelink` zones missing | Add all three to the DNS zone group |
| Search indexer cannot read storage | Service-to-service path: no shared private link, trusted-service exception or perimeter membership | Configure one (file 18) |
| Spoke VM cannot reach another spoke | Peering is not transitive; no route to the hub firewall | UDR and firewall rule |
| On-premises can reach a spoke, replies vanish | No route on `GatewaySubnet` for the spoke prefix to the firewall | Add it |
| New VM cannot reach the internet | Private subnet, no outbound method | NAT gateway or firewall route |
| Image pull fails in a locked-down subnet | Registry or `mcr.microsoft.com` not allowed; or ACR private endpoint missing the data endpoint record | Firewall rules or complete the private DNS |
| Application Gateway returns 502 | Backend unhealthy: NSG, DNS, host name or certificate | *Backend health* blade states the reason |
| Deployment pipeline cannot reach private resources | Hosted agents run outside the network | VNet-connected agents |
| Role assigned, still 403 | Propagation delay, wrong scope, wrong principal (app ID vs object ID), cached token | Wait, verify assignment, get a new token |
| Intermittent timeouts under load | SNAT exhaustion | Reuse connections, private endpoints, NAT gateway |
| Streaming responses arrive all at once or cut off | Proxy buffering or timeout | Disable buffering, raise timeouts |

### Flow logs for evidence

When you need proof of what happened:

```kusto
// Denied flows to a destination in the last hour (Traffic Analytics)
NTANetAnalytics
| where TimeGenerated > ago(1h) and FlowStatus == "Denied"
| where DestIp == "10.20.3.5"
| project TimeGenerated, SrcIp, DestIp, DestPort, L4Protocol, AclRule, FlowDirection
| order by TimeGenerated desc
```

Table and column names depend on the Traffic Analytics schema version in use. The `AclRule` (or equivalent) column names the rule that denied the flow.

### Proactive monitoring

Do not wait for a ticket:

- **Connection Monitor** tests from the app subnet to each critical dependency (model endpoint, search, database, on-premises system) on its real port, with alerts on reachability and latency.
- **Alerts** on gateway BGP and tunnel state, firewall health and SNAT utilisation, Application Gateway unhealthy host count.
- **Synthetic availability tests** in Application Insights for the user-facing endpoint.
- A **DNS check** in the application's health endpoint or start-up log: resolve each dependency and log the IP. It makes the most common fault self-evident.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Flow logs on everything | VNet flow logs on production networks. Use sampling, retention limits and Traffic Analytics intervals to manage cost |
| Connection Monitor coverage | Critical paths only: hybrid links and key dependencies |
| Packet capture | A last resort for protocol-level problems. Captures can contain sensitive data; limit duration, filter tightly and protect the output |
| Diagnostic tooling in images | Include basic tools (`curl`, `dig`, `nc`) in base images or have a debug sidecar ready. A locked-down container with no tools slows every incident |
| Who can run diagnostics | Network Watcher actions need specific permissions. Give workload teams enough (for example Network Watcher read and diagnostic actions on their resources) to self-serve |

## Hands-on

```bash
RG=rg-nw-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Two subnets, two VMs, no public IPs, with a NAT gateway for package installs
az network vnet create -g $RG -n vnet-nw --address-prefixes 10.70.0.0/16 -o none
az network public-ip create -g $RG -n pip-ng --sku Standard -o none
az network nat gateway create -g $RG -n ng-nw --public-ip-addresses pip-ng -o none
for S in a b; do
  az network nsg create -g $RG -n nsg-$S -o none
  az network vnet subnet create -g $RG --vnet-name vnet-nw -n snet-$S \
    --address-prefixes 10.70.$([ $S = a ] && echo 1 || echo 2).0/24 --nsg nsg-$S --nat-gateway ng-nw -o none
  az vm create -g $RG -n vm-$S --image Ubuntu2404 --size Standard_B2s --vnet-name vnet-nw --subnet snet-$S \
    --public-ip-address "" --nsg "" --admin-username azureuser --generate-ssh-keys -o none
done
IPB=$(az vm list-ip-addresses -g $RG -n vm-b --query "[0].virtualMachine.network.privateIpAddresses[0]" -o tsv)
IPA=$(az vm list-ip-addresses -g $RG -n vm-a --query "[0].virtualMachine.network.privateIpAddresses[0]" -o tsv)

# 2. Start a listener on vm-b
az vm run-command invoke -g $RG -n vm-b --command-id RunShellScript \
  --scripts "nohup python3 -m http.server 8080 >/dev/null 2>&1 &" -o none

# 3. Baseline: works (default AllowVnetInBound)
az vm run-command invoke -g $RG -n vm-a --command-id RunShellScript \
  --scripts "curl -s -m 5 -o /dev/null -w '%{http_code}\n' http://$IPB:8080" --query "value[0].message" -o tsv

# 4. Break it with an NSG rule
az network nsg rule create -g $RG --nsg-name nsg-b -n DenyFromA --priority 200 --direction Inbound \
  --access Deny --protocol Tcp --source-address-prefixes 10.70.1.0/24 --destination-port-ranges 8080 -o none

# 5. Diagnose without logging in
az network watcher test-ip-flow -g $RG --vm vm-b --direction Inbound --protocol TCP \
  --local $IPB:8080 --remote $IPA:50000
az network watcher show-next-hop -g $RG --vm vm-a --source-ip $IPA --dest-ip $IPB
az network watcher test-connectivity -g $RG --source-resource vm-a --dest-address $IPB --dest-port 8080 \
  --query "{status:connectionStatus, hops:hops[].{type:type, issues:issues}}"

# 6. Second fault: remove the deny, stop the listener, and compare "refused" with "timed out"
az network nsg rule delete -g $RG --nsg-name nsg-b -n DenyFromA
az vm run-command invoke -g $RG -n vm-b --command-id RunShellScript --scripts "pkill -f http.server" -o none
az vm run-command invoke -g $RG -n vm-a --command-id RunShellScript \
  --scripts "curl -sv -m 5 http://$IPB:8080 2>&1 | tail -3" --query "value[0].message" -o tsv

az group delete -n $RG --yes --no-wait
```

`test-connectivity` installs the Network Watcher agent extension on first use. Note how step 5 names the exact rule, and how step 6's error differs from a timeout.

Further exercises:

1. Add a route table sending `10.70.2.0/24` to a non-existent appliance and diagnose it with *next hop*.
2. In the portal, create a **Connection Monitor** test from `vm-a` to `https://learn.microsoft.com` and view the path.
3. Write a one-page runbook for "application cannot reach its model endpoint" using the method above.

## Security, cost and reliability notes

- **Security:** flow logs and packet captures contain sensitive metadata and sometimes payloads. Restrict access, set retention, and store in protected accounts.
- **Security:** diagnostic permissions (packet capture, Run Command) are powerful. Scope and audit them.
- **Cost:** flow logs cost storage; Traffic Analytics adds Log Analytics ingestion by volume; Connection Monitor is billed per test. Tune intervals and scope.
- **Reliability:** rehearse diagnosis before an incident. Knowing the tools exist is not the same as being quick with them.

## Common mistakes

- Starting with firewall rules when the name does not resolve correctly.
- Using `ping` and concluding the network is down.
- Testing from your laptop instead of from the failing source.
- Treating every 403 as RBAC, or every 403 as network.
- Changing several things at once and not knowing which fixed it.
- Checking only one direction of the path.
- Forgetting security admin rules, service firewalls and network security perimeters when the NSGs look fine.
- No flow logs enabled when evidence is needed.
- Containers with no diagnostic tools and no debug path.
- Escalating to Microsoft support without request IDs, timestamps and the results of basic tests.

## Check yourself

1. List the layers a connection must pass, in order.
2. What does a timeout suggest, compared with a connection refused?
3. Which Network Watcher tool tells you which NSG rule blocks a flow?
4. Which tool tells you where a packet will be sent next?
5. A private Foundry endpoint returns 403 to an app with the correct role. What do you check first, and how?
6. Why is `ping` a poor connectivity test in Azure?
7. What replaces NSG flow logs?
8. Name three proactive checks that catch network problems before users do.

<details>
<summary>Answers</summary>

1. DNS, routing, filtering, listener, TLS, authentication, authorisation, application.
2. A timeout suggests packets are being dropped silently (filtering, routing, unreachable). A refusal means the host was reached but nothing is listening on that port, or something actively rejected the connection.
3. IP flow verify (or NSG diagnostics).
4. Next hop.
5. DNS: resolve the endpoint name from the app and confirm it returns the private IP, since a public IP means the request reached the service on a path it rejects.
6. Many services and load balancers do not respond to ICMP and NSGs often block it, so failure proves little. Test the actual TCP port.
7. Virtual network flow logs.
8. Any three of: Connection Monitor tests to key dependencies, alerts on gateway, firewall and Application Gateway health, synthetic availability tests, logging resolved dependency IPs at start-up.

</details>

## References

- [What is Azure Network Watcher?](https://learn.microsoft.com/azure/network-watcher/network-watcher-overview)
- [IP flow verify overview](https://learn.microsoft.com/azure/network-watcher/ip-flow-verify-overview)
- [Next hop overview](https://learn.microsoft.com/azure/network-watcher/next-hop-overview)
- [Connection troubleshoot overview](https://learn.microsoft.com/azure/network-watcher/connection-troubleshoot-overview)
- [Connection monitor overview](https://learn.microsoft.com/azure/network-watcher/connection-monitor-overview)
- [Virtual network flow logs](https://learn.microsoft.com/azure/network-watcher/vnet-flow-logs-overview)
- [Packet capture overview](https://learn.microsoft.com/azure/network-watcher/packet-capture-overview)
- [Troubleshoot Azure Private Endpoint connectivity problems](https://learn.microsoft.com/azure/private-link/troubleshoot-private-endpoint-connectivity)
- [Diagnose a virtual machine network traffic filter problem](https://learn.microsoft.com/azure/virtual-network/diagnose-network-traffic-filter-problem)
- [Troubleshoot outbound connectivity with Azure Firewall logs](https://learn.microsoft.com/azure/firewall/firewall-structured-logs)
