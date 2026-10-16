# Azure Bastion and Secure Admin Access

> Topic: 05 · Networking · Tracks: E ● A ● C – · Status checked: 2026-10

## Why it matters

Internet-exposed RDP and SSH ports are scanned within minutes of appearing and remain one of the most common ways attackers get in. Once an environment is network-isolated, the opposite problem arises: how do legitimate people reach anything? This file covers the safe ways in. For AI work it has a very practical edge, because a private Foundry environment is unusable from a laptop on the internet until you have an access path into the network.

## Key concepts

### The principle

Management ports (22, 3389, and their kin) are **never** open to the internet. Administrative access is:

- **brokered** through a hardened service;
- **authenticated** with Entra ID and MFA;
- **authorised** just-in-time and least-privilege;
- **logged**.

### Azure Bastion

A managed PaaS jump service deployed into a virtual network. You connect to Bastion over TLS on port 443, and Bastion connects to the target VM's **private IP** over RDP or SSH. Target VMs need no public IP.

```text
Admin (browser or native client)
      │  HTTPS 443, Entra ID sign-in + MFA
      ▼
Azure Bastion (in AzureBastionSubnet)
      │  RDP 3389 / SSH 22 to private IP
      ▼
VM in the same or a peered VNet
```

**SKUs**

| | **Developer** | **Basic** | **Standard** | **Premium** |
|---|---|---|---|---|
| Cost | Free | Hourly | Hourly, higher | Hourly, highest |
| Deployment | Shared pool, no dedicated host or subnet | Dedicated, in `AzureBastionSubnet` | Dedicated | Dedicated |
| Reaches VMs in | The same VNet only | Same and peered VNets | Same and peered VNets | Same and peered VNets |
| Concurrent connections | One at a time | Fixed (2 instances) | Scales with **host scaling** (instance count) | Scales |
| Native client (`az network bastion ssh/rdp/tunnel`) | No | No | Yes | Yes |
| File transfer, custom ports, IP-based connection, shareable links, Kerberos | No | Limited | Yes | Yes |
| **Session recording** | No | No | No | Yes |
| **Private-only deployment** (no public IP on Bastion) | No | No | No | Yes |

Developer is for development and test; it is regionally limited and has no SLA. Use Standard or Premium for production.

**Requirements**

- A subnet named **`AzureBastionSubnet`**, `/26` or larger (not needed for Developer).
- A Standard public IP (except private-only Premium).
- If you attach an NSG to the Bastion subnet, the documented inbound and outbound rules must all be present, or Bastion breaks. Many teams leave the Bastion subnet's NSG to exactly that rule set.
- No UDR with a default route on the Bastion subnet.
- On the **target** subnet: allow inbound 22/3389 from the Bastion subnet's range.

**Roles needed to connect:** Reader on the VM, Reader on its NIC, Reader on the Bastion resource (and on the VNet when crossing peerings). Plus a way to authenticate to the VM itself.

**Features worth knowing**

| Feature | Use |
|---|---|
| **Native client support** | Use your own SSH or RDP client through a Bastion tunnel, including `az network bastion tunnel` for port forwarding to non-VM targets such as a database or a private web UI |
| **IP-based connection** | Reach on-premises or non-Azure machines over ExpressRoute or VPN |
| **Shareable links** | A URL to connect to a VM without portal access. Use with caution; treat links as credentials |
| **Session recording** (Premium) | Records graphical sessions to a storage account for audit |
| **Private-only** (Premium) | Bastion reachable only over private connectivity, for environments with no public entry at all |
| **Entra ID authentication** | Sign in to Linux (SSH) and Windows (RDP) VMs with Entra identities, covered below |
| **AKS and private cluster access** | Tunnel to a private API server |

### Authenticating to the VM

Bastion gets you a network path. You still log in to the operating system.

| Method | Notes |
|---|---|
| **Microsoft Entra ID login** | The VM has the Entra login extension; users need *Virtual Machine Administrator Login* or *Virtual Machine User Login*. No local passwords, MFA and Conditional Access apply, access ends when the role is removed. **Preferred** |
| **SSH keys** (Linux) | Keys from Key Vault or local files. Better: short-lived SSH certificates issued through Entra ID with `az ssh` |
| **Local username and password** | Avoid. If unavoidable, store in Key Vault and rotate |
| **Domain credentials** | For domain-joined Windows servers |

### Just-in-time VM access

A Microsoft Defender for Servers feature. Management ports are **closed** by an NSG deny rule (or Azure Firewall rule). A user with permission requests access for a specific VM, port, source IP and duration; Defender adds a temporary allow rule and removes it when the time expires. Requests are logged.

Bastion and JIT are complementary: JIT can open the path from the Bastion subnet only when needed.

### Other access paths

| Method | What it is | Fits |
|---|---|---|
| **Point-to-site VPN** with Entra authentication | The device joins the virtual network | Developers and admins needing broad private network access; apply Conditional Access |
| **Microsoft Entra Private Access** | Per-application Zero Trust network access | Replacing VPN with least-privilege access (topic 02, file 11) |
| **Jump box VM** | A hardened VM inside the network, reached through Bastion | A place to run tools with line of sight to private endpoints |
| **Windows 365 / Azure Virtual Desktop in the VNet** | A managed workstation already inside the boundary | Developer workstations for private environments (Dev Box is in maintenance mode; Windows 365 is the path forward) |
| **Run Command / serial console** | Execute scripts on, or get console access to, a VM through the Azure control plane without network connectivity | Break-glass and automation. Governed by RBAC; logged in the activity log |
| **Azure Arc SSH** | SSH to Arc-enabled servers through the Arc agent, no inbound ports | Hybrid servers |
| **Self-hosted build agents in the VNet** | Pipelines with network line of sight | Deployments to private resources; Managed DevOps Pools or GitHub runners injected into a VNet |

### Privileged access workstations

For the most sensitive administration, the **device** matters as much as the path. A privileged access workstation (PAW) is a hardened, dedicated device used only for administration, with no email or general browsing. Conditional Access can require a compliant PAW for privileged roles and the Azure management endpoints. It closes the gap where a compromised everyday laptop becomes a compromised admin session.

### Do you need to log in at all?

The best admin access is none. Each of these removes a reason to connect interactively:

| Instead of logging in to… | Use |
|---|---|
| Patch servers | Azure Update Manager |
| Change configuration | IaC, machine configuration, immutable images |
| Read logs | Azure Monitor, Log Analytics |
| Run a one-off command | Run Command, with RBAC and logging |
| Debug an application | Application Insights, container console through RBAC |
| Deploy | Pipelines with a workload identity |
| Manage VMs at all | PaaS and serverless hosting |

## How it works on Azure

### Placement

| Topology | Where Bastion goes |
|---|---|
| Hub-and-spoke | In the **hub**, serving all peered spokes |
| Virtual WAN | In a **shared services spoke** (it cannot live in the virtual hub) |
| Standalone workload | In the workload VNet, or Developer SKU for dev/test |

One Standard or Premium Bastion per regional hub serves every spoke peered to it. Scale instances for concurrent sessions.

### Access to a private AI environment

A network-isolated Foundry workload has no public endpoints, so the Foundry portal in your browser cannot reach the project's data plane, and your laptop cannot call the model or search endpoints. Options:

| Persona | Recommended path |
|---|---|
| **Developers** building and testing | A workstation inside the network: Windows 365 or AVD in the VNet, or a jump box through Bastion; or P2S VPN / Private Access with DNS configured for private zones |
| **Operators** investigating an issue | Bastion to a jump box with tooling installed; or Azure Monitor and Foundry tracing with no network access needed |
| **Pipelines** deploying code and running evaluations | Build agents with VNet line of sight |
| **End users** of the application | Not an admin path: Application Gateway or Front Door to the app |

Whichever you choose, **DNS must follow**. A VPN client that reaches the VNet but resolves `*.openai.azure.com` through public DNS gets a public IP and is refused. P2S clients need the VNet's DNS servers pushed to them, or the client must use the DNS Private Resolver.

The reference architecture for the baseline Foundry chat application uses Bastion plus a jump box for exactly this reason. In non-production, some teams allow selected developer IP addresses through the resource firewalls instead; that is a pragmatic compromise to make explicitly, not a default.

### Using Bastion from the command line

```bash
# SSH with Entra ID authentication through Bastion (Standard or Premium SKU)
az network bastion ssh -g rg-hub -n bas-hub-swc \
  --target-resource-id $VM_ID --auth-type AAD

# RDP with the native client (Windows)
az network bastion rdp -g rg-hub -n bas-hub-swc --target-resource-id $VM_ID

# Port-forward to a service on the VM (for example a notebook or admin UI on 8888)
az network bastion tunnel -g rg-hub -n bas-hub-swc \
  --target-resource-id $VM_ID --resource-port 8888 --port 8888
# then browse to http://localhost:8888
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Bastion or VPN | Bastion for VM administration with no client software and per-VM RBAC. VPN or Private Access when people need to reach many private services from their own tools |
| Which Bastion SKU | Developer for dev/test in one VNet. Standard for production. Premium when session recording or a private-only entry is required |
| Central or per-workload Bastion | Central in the hub. Per-workload only for isolated environments |
| Entra login or local accounts | Entra login. It removes credential management and brings Conditional Access |
| JIT | Yes for any VMs with management ports, where Defender for Servers is licensed |
| Jump box or managed workstation | A managed workstation (Windows 365, AVD) gives a better developer experience for sustained work; a jump box is cheaper for occasional use |
| Shareable links | Avoid in production |

**Cost.** Standard and Premium Bastion bill hourly per instance plus outbound data, whether anyone is connected or not. It is a platform cost, shared across workloads. Developer SKU is free with strict limits.

## Hands-on

```bash
RG=rg-bastion-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Network and a VM with no public IP, Entra login enabled
az network vnet create -g $RG -n vnet-bas --address-prefixes 10.60.0.0/16 \
  --subnet-name snet-vm --subnet-prefixes 10.60.1.0/24 -o none
az vm create -g $RG -n vm-private --image Ubuntu2404 --size Standard_B2s \
  --vnet-name vnet-bas --subnet snet-vm --public-ip-address "" \
  --assign-identity --admin-username azureuser --generate-ssh-keys -o none
az vm extension set -g $RG --vm-name vm-private --publisher Microsoft.Azure.ActiveDirectory --name AADSSHLoginForLinux -o none
VM_ID=$(az vm show -g $RG -n vm-private --query id -o tsv)
az role assignment create --assignee-object-id $(az ad signed-in-user show --query id -o tsv) \
  --assignee-principal-type User --role "Virtual Machine Administrator Login" --scope $VM_ID -o none

# 2a. Free option: Developer SKU (if available in your region), then connect from the portal
az network bastion create -g $RG -n bas-dev --vnet-name vnet-bas --sku Developer -l $LOC -o none

# 2b. Or Standard SKU with native client support (billable; ~10 minutes to deploy)
# az network vnet subnet create -g $RG --vnet-name vnet-bas -n AzureBastionSubnet --address-prefixes 10.60.0.0/26 -o none
# az network public-ip create -g $RG -n pip-bas --sku Standard -o none
# az network bastion create -g $RG -n bas-std --vnet-name vnet-bas --public-ip-address pip-bas \
#    --sku Standard --enable-tunneling true -l $LOC -o none
# az extension add -n bastion -n ssh 2>/dev/null
# az network bastion ssh -g $RG -n bas-std --target-resource-id $VM_ID --auth-type AAD

# 3. Run a command with no network path at all
az vm run-command invoke -g $RG -n vm-private --command-id RunShellScript \
  --scripts "hostname; whoami; ss -tlnp | head" --query "value[0].message" -o tsv

# 4. Confirm the VM has no public IP and find the session in the logs
az vm list-ip-addresses -g $RG -n vm-private --query "[0].virtualMachine.network" 
az monitor activity-log list -g $RG --offset 1h --query "[?contains(operationName.value,'runCommand')].{op:operationName.value, by:caller}" -o table

az group delete -n $RG --yes --no-wait
```

In the portal: open the VM, choose **Connect → Bastion**, and sign in with Microsoft Entra ID. Then review *Bastion → Sessions*.

If you have Defender for Servers, enable JIT on the VM, request access, and watch the NSG rule appear and disappear.

## Security, cost and reliability notes

- **Security:** enforce by policy: no public IPs on NICs, no NSG rules allowing management ports from the internet.
- **Security:** Bastion reduces exposure; it does not replace hardening the VM, MFA, or least privilege.
- **Security:** a jump box is a high-value target. Harden it, patch it, restrict who can log in, and give it no more network reach than its purpose needs.
- **Security:** Run Command is powerful (it runs as root or SYSTEM). Restrict the `runCommand` action and alert on its use.
- **Security:** Premium session recording produces sensitive data; protect the storage account.
- **Cost:** a dedicated Bastion bills continuously. Share one per hub.
- **Reliability:** Bastion is zone-redundant in supported regions with appropriate configuration. Keep a second path (Run Command, serial console) for when it is unavailable.

## Common mistakes

- Public IPs and open 22 or 3389 "just for now".
- Wrong subnet name or size for Bastion.
- An NSG or UDR on the Bastion subnet that breaks it.
- Forgetting the target subnet's NSG must allow the Bastion subnet.
- Local admin passwords shared in a wiki when Entra login was available.
- Expecting the Developer SKU to reach peered VNets or handle several sessions.
- A VPN that connects but resolves private endpoints through public DNS.
- Building a private AI environment with no thought for how developers will work in it.
- Leaving JIT off and management ports permanently open inside the network.
- Deploying a Bastion per spoke.

## Check yourself

1. How does Azure Bastion avoid exposing a VM's management ports?
2. Which Bastion SKU supports the native client and tunnelling? Which adds session recording?
3. What does just-in-time VM access do?
4. Why is Entra ID login to VMs preferred over local accounts?
5. A developer on P2S VPN gets a 403 from a private Foundry endpoint. What is the likely cause?
6. Where does Bastion go in a Virtual WAN design?
7. Name three ways to avoid interactive logins to servers.

<details>
<summary>Answers</summary>

1. The administrator connects to Bastion over HTTPS, and Bastion connects to the VM's private IP from inside the virtual network, so the VM needs no public IP or internet-facing management port.
2. Standard (and Premium) support native client and tunnelling. Premium adds session recording and private-only deployment.
3. Keeps management ports closed and opens them for an approved requester, source, port and limited time, then closes them again, with logging.
4. No local credentials to manage or leak, MFA and Conditional Access apply, access is governed by RBAC and can use PIM, and it ends when the role is removed.
5. DNS: the client resolves the endpoint's public address because it is not using the VNet's DNS or private resolver.
6. In a shared services spoke, since it cannot be deployed in the virtual hub.
7. Any three of: Update Manager for patching, IaC and immutable images for configuration, central logging, Run Command, pipelines for deployment, PaaS hosting.

</details>

## References

- [What is Azure Bastion?](https://learn.microsoft.com/azure/bastion/bastion-overview)
- [About Azure Bastion configuration settings and SKUs](https://learn.microsoft.com/azure/bastion/configuration-settings)
- [Working with NSG access and Azure Bastion](https://learn.microsoft.com/azure/bastion/bastion-nsg)
- [Connect to a VM using a native client](https://learn.microsoft.com/azure/bastion/native-client)
- [Sign in to a Linux VM by using Microsoft Entra ID and OpenSSH](https://learn.microsoft.com/entra/identity/devices/howto-vm-sign-in-azure-ad-linux)
- [Sign in to a Windows VM by using Microsoft Entra ID](https://learn.microsoft.com/entra/identity/devices/howto-vm-sign-in-azure-ad-windows)
- [Understand just-in-time VM access](https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-overview)
- [Run scripts in your VM by using Run Command](https://learn.microsoft.com/azure/virtual-machines/run-command-overview)
- [Privileged access workstations](https://learn.microsoft.com/security/privileged-access-workstations/privileged-access-devices)
- [Plan for landing zone network segmentation and remote access](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/plan-for-virtual-machine-remote-access)
