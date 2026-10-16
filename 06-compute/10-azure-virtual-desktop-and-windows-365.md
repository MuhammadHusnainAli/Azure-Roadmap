# Azure Virtual Desktop and Windows 365

> Topic: 06 · Compute · Tracks: E – A ○ C ○ · Status checked: 2026-10

## Why it matters

Desktop virtualisation delivers a Windows desktop or individual applications from the cloud to any device. It is a large Azure workload in its own right, and it intersects with AI projects in a specific way: when an AI environment is network-isolated, developers and operators need a workstation *inside* the network, and a cloud desktop is often the cleanest answer. Consultants also meet it constantly in remote-work, contractor, call-centre and regulated-industry engagements. The product line shifted in 2025 and 2026, so check current status.

## Key concepts

### The two services

| | **Azure Virtual Desktop (AVD)** | **Windows 365** |
|---|---|---|
| Model | A **platform** you design and operate on Azure infrastructure | A **SaaS Cloud PC**, provisioned for you |
| Pricing | Consumption: you pay for the VMs, storage and networking, plus user access rights | **Fixed per user per month** for a chosen size |
| Desktop type | **Pooled** (many users share hosts) or **personal** (one user per VM) | **Personal**, persistent Cloud PC; shared and frontline modes exist |
| Multi-session Windows | **Yes**: Windows 11 Enterprise multi-session | No, single user per Cloud PC |
| Publish individual apps | Yes (RemoteApp) | Limited (Cloud Apps for frontline) |
| Management | Azure portal, plus Intune or other tools for the session hosts | **Microsoft Intune**, the same as physical PCs |
| Control and flexibility | High: any VM size, image, scaling logic, GPU | Lower: fixed size options, opinionated |
| Operational effort | Higher | Low |
| Cost at low usage | Low, with autoscale and pooling | Fixed regardless of use |
| Best for | Cost-optimised shared desktops, published apps, variable usage, complex or specialised requirements | Simple, predictable personal desktops managed like PCs |

Both use the same remote desktop protocol technology and the same **Windows App** client, and can be used side by side.

### Azure Virtual Desktop architecture

Microsoft runs the **control plane**; you run the **session hosts**.

```text
User (Windows App / browser)
        │  HTTPS, reverse connect
Microsoft-managed: Web · Gateway · Broker · Diagnostics · Load balancing
        │
Your subscription: Host pool → Session hosts (VMs) in your VNet
                   FSLogix profiles on Azure Files / Azure NetApp Files
                   Identity: Microsoft Entra ID (and AD DS where needed)
```

| Object | Meaning |
|---|---|
| **Host pool** | A collection of identical session host VMs. **Pooled** (users are load-balanced across hosts) or **personal** (each user assigned a VM) |
| **Session host** | A VM running Windows 11 or Windows 10 Enterprise (single or **multi-session**), or Windows Server, with the AVD agent |
| **Application group** | What is published: a full **desktop**, or **RemoteApp** applications |
| **Workspace** | A grouping of application groups shown to users |
| **Scaling plan** | Schedules and rules that start, stop and drain session hosts |

**Reverse connect** means session hosts make outbound connections to the service. No inbound ports are opened on your network.

**Load balancing** for pooled host pools: *breadth-first* spreads users across hosts for performance; *depth-first* fills one host before using the next, so idle hosts can be shut down to save cost.

### Profiles

In a pooled environment a user may land on a different host each time, so their profile must follow them. **FSLogix profile containers** store the profile in a VHDX file on an SMB share, attached at sign-in.

| Storage | Notes |
|---|---|
| **Azure Files** (premium) | The usual choice. Identity-based access with Entra Kerberos or AD DS |
| **Azure NetApp Files** | Higher performance and scale |

Profile storage performance directly affects sign-in time. Size it and place it in the same region as the hosts.

### Identity and access

- Session hosts can be **Microsoft Entra joined** (cloud-only, the target state), hybrid joined, or AD DS domain joined.
- Users authenticate with Entra ID. **Conditional Access** applies: MFA, compliant device, location.
- **Single sign-on** to the session with Entra authentication, including passwordless.
- RBAC roles control who can manage host pools and who is assigned to application groups.

### Features worth knowing

| Feature | Purpose |
|---|---|
| **Autoscale (scaling plans)** | Power hosts on and off by schedule and demand, for pooled and personal pools; **dynamic autoscaling** can also create and delete hosts |
| **RDP Shortpath** | Direct UDP transport between client and host for lower latency, over private networks or the public internet |
| **App attach** | Deliver applications dynamically from packages without installing them in the image |
| **Session host update / host pool management** | Roll new images across a pool in a controlled way |
| **Regional host pools** | Host pool metadata kept within a chosen region, reducing cross-region dependencies |
| **Private Link** | Private connectivity to the AVD service |
| **GPU session hosts** | NV-series for graphics-intensive work |
| **Watermarking, screen capture protection, clipboard and drive redirection controls** | Data-leak prevention |
| **Azure Virtual Desktop Insights** | Monitoring workbook on Azure Monitor |
| **AVD on Azure Local / Azure Virtual Desktop Hybrid** | Session hosts in your own data centre or edge location with the Azure control plane, for data residency and latency |

### Licensing

Users need an eligible licence: Microsoft 365 E3, E5, Business Premium, or Windows Enterprise E3/E5 (per user) for Windows client desktops; RDS CALs with Software Assurance for Windows Server. For external commercial users (customers, not employees) there is **per-user access pricing**. The Azure infrastructure is charged separately.

### Windows 365

A Cloud PC is a persistent Windows 11 VM dedicated to a user, provisioned automatically when a licence is assigned.

| Edition | For |
|---|---|
| **Enterprise** | Organisations managing devices with Intune. Custom images, own-network connectivity, full policy control |
| **Business** | Small organisations, simpler setup, limited management |
| **Frontline** | Shift and part-time workers: licences are shared across users who are not concurrently signed in, in dedicated or shared modes |
| **Government** | US government clouds |

Features: provisioning policies and images, **Microsoft-hosted network** or an **Azure network connection** to your own VNet, point-in-time restore, resize, Windows 365 Boot and Switch for device integration, **Windows 365 Link** thin-client devices, GPU-enabled sizes, and a developer-oriented configuration.

### Microsoft Dev Box is in maintenance mode

Dev Box, the developer workstation service, **stopped accepting new customers on 1 November 2025** and is in **maintenance mode with no new features planned**. Microsoft's stated path forward for virtual developer environments is **Windows 365**, which has been gaining developer-focused capabilities (developer images, more sizes including GPU). Existing Dev Box customers can continue to use and scale current deployments; they should plan a transition and watch for retirement announcements.

For new work, do not design around Dev Box. Use Windows 365 for developer Cloud PCs, AVD where you need more control, or dev containers and GitHub Codespaces for Linux-first development.

## How it works on Azure

### Choosing

```text
Do users need a full personal Windows PC, managed like any other device, at a predictable cost?
   └─ Yes → Windows 365

Do many users need the same standard desktop or a few applications, with variable hours?
   └─ Yes → Azure Virtual Desktop, pooled, multi-session, autoscaled

Do you need specific VM sizes, GPUs, custom scaling, published RemoteApps, or desktops on your own hardware?
   └─ Yes → Azure Virtual Desktop

Is it only about administrators reaching servers?
   └─ No desktop service needed → Azure Bastion (topic 05, file 14)
```

Cost intuition: Windows 365 costs the same whether the Cloud PC is used one hour or two hundred hours a month. AVD pooled desktops share hosts and switch them off when idle, so cost per user is lower for many users with standard needs, in exchange for design and operations effort.

### Cloud desktops for private AI development

A network-isolated Foundry environment cannot be reached from a developer's laptop on the internet (topic 05, file 18). A cloud desktop inside the network solves several problems at once:

| Need | How a Cloud PC or AVD desktop helps |
|---|---|
| Reach private endpoints (Foundry portal data plane, search, storage) | The desktop sits in a VNet with line of sight and correct DNS |
| Keep sensitive data off personal devices | Data stays in the session; clipboard, drive and print redirection can be restricted |
| Consistent, pre-configured tooling | Image with VS Code, CLI tools, SDKs, container runtime |
| Contractors and partners | Access to a governed workstation without issuing hardware |
| Conditional Access and audit | Entra sign-in with MFA and device requirements; session logging |
| GPU for local experimentation | GPU-enabled sizes |

Design points:

- Connect the Cloud PC (Azure network connection) or AVD hosts to a **subnet with DNS that resolves the private zones**.
- Apply **egress control** as for any workload subnet.
- Decide **data movement policy**: whether clipboard and file transfer are allowed in each direction.
- For Linux-centric development, combine with **WSL**, dev containers, or a Linux jump host; cloud desktops are Windows.

Alternatives for the same need: Bastion with a jump box, point-to-site VPN or Entra Private Access from a managed laptop.

### Sizing AVD

- Classify users: light (data entry), medium (office), heavy (developers, analysts), power (graphics).
- Use Microsoft's users-per-vCPU guidance per class as a starting point for multi-session hosts, then **measure** with a pilot.
- Prefer several mid-sized hosts over a few very large ones, for resilience and smoother scaling.
- Spread hosts across **availability zones**.
- Size profile storage for IOPS at sign-in peaks.

### Images and updates

Session hosts are a classic golden-image workload (file 05): build an image with applications and optimisations, publish to a Compute Gallery, and roll it across host pools with session host update or by replacing hosts. Pooled hosts should be treated as disposable.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Windows 365 or AVD | Windows 365 for simplicity and predictable per-user cost. AVD for pooled economics, RemoteApp, and control |
| Pooled or personal (AVD) | Pooled for standardised task workers. Personal for users needing admin rights, persistent installs or dedicated resources |
| Entra join or domain join | Entra join where applications allow it |
| Profile storage | Azure Files premium by default; NetApp Files at scale |
| Breadth-first or depth-first | Depth-first with autoscale for cost; breadth-first for the best experience |
| Region | Close to users for latency. Close to the data and applications they use |
| Data-leak controls | Restrict redirections according to data sensitivity. Tight controls reduce convenience; agree them with the business |
| Cloud desktop for AI developers | A good default for isolated environments. Budget for it |

## Hands-on

Deploying AVD or Windows 365 needs licences and takes time. Explore instead.

1. In the Azure portal, open **Azure Virtual Desktop → Create a host pool**. Walk through the wizard: pool type, load-balancing algorithm, session host image and size, domain join options (Entra ID), workspace registration. Note the options and cancel.
2. Open **Scaling plans → Create** and read the ramp-up, peak, ramp-down and off-peak schedule model.
3. In the **Microsoft Intune admin centre**, open **Devices → Windows 365** (if your tenant has it) and review *Provisioning policies* and *Azure network connection*.
4. Read the Windows 365 size and price list, and price an AVD pooled design for the same users.

Costing exercise: 200 call-centre agents work eight-hour shifts, five days a week, using a browser and a softphone. 20 developers need a persistent, powerful workstation with admin rights inside a private network.

- Propose a service and configuration for each group.
- Estimate monthly cost for the agents two ways: Windows 365 per user, and AVD pooled multi-session with depth-first autoscale (assume a users-per-vCPU figure and compute hours actually used).
- List the data-leak controls you would enable for each group.

<details>
<summary>Notes</summary>

Agents: AVD pooled, Windows 11 multi-session, depth-first with a scaling plan matching shifts, FSLogix on Azure Files, restricted redirection; or Windows 365 Frontline if shift patterns suit licence sharing. Developers: Windows 365 Enterprise on an Azure network connection into the workload VNet, or AVD personal desktops for more size and GPU choice. AVD pooled typically costs less per agent because hosts are shared and powered off outside shifts; Windows 365 trades some cost for much lower operational effort.

</details>

## Security, cost and reliability notes

- **Security:** require MFA and Conditional Access for desktop access. A cloud desktop is a door into your network.
- **Security:** session hosts are endpoints. Patch them, run Defender for Endpoint, and manage them with Intune or equivalent.
- **Security:** decide clipboard, drive, printer and USB redirection per user group. Use watermarking and screen capture protection for sensitive data.
- **Cost:** AVD cost is dominated by session host compute. Autoscale, depth-first load balancing, right-sizing, reservations or savings plans for the steady base, and deallocating personal desktops when idle.
- **Cost:** Windows 365 is predictable but pays for idle time; Frontline reduces that for shift workers.
- **Reliability:** spread hosts across zones; make profile storage zone-redundant; plan a secondary region for critical desktops, including profile replication.
- **Reliability:** the user experience depends on network latency to the region. Test from where users actually are.

## Common mistakes

- Choosing AVD for a small group who needed simple personal PCs, and carrying the operational overhead.
- Choosing Windows 365 for hundreds of part-time task workers and paying for idle Cloud PCs.
- Session hosts left running around the clock.
- Under-provisioned profile storage, giving slow sign-ins.
- Hand-maintained session hosts instead of image-based replacement.
- Wide-open clipboard and drive redirection in a data-sensitive environment.
- Desktops in a region far from users or from the applications they use.
- Planning new developer environments on Dev Box.
- A private AI environment delivered with no workstation strategy for the people who must work in it.

## Check yourself

1. Give four differences between Azure Virtual Desktop and Windows 365.
2. What is the difference between pooled and personal host pools?
3. What problem do FSLogix profile containers solve?
4. What does reverse connect mean for network security?
5. What is the current status of Microsoft Dev Box, and what replaces it?
6. Why is a cloud desktop useful for teams working on a network-isolated AI environment?
7. Which AVD settings most affect cost?

<details>
<summary>Answers</summary>

1. AVD is a platform you operate and Windows 365 is a SaaS Cloud PC; AVD is consumption-priced and Windows 365 is fixed per user; AVD supports pooled multi-session desktops and RemoteApp while Windows 365 is personal; AVD offers more control over sizes, scaling and images with more operational effort.
2. Pooled: users are load-balanced across shared hosts and may get a different host each session. Personal: each user is assigned a dedicated VM.
3. A user's profile follows them to whichever host they land on, by attaching a profile disk from shared storage at sign-in.
4. Session hosts make outbound connections to the AVD service, so no inbound ports need to be opened to them.
5. It is in maintenance mode with no new features and has not accepted new customers since 1 November 2025. Windows 365 is the recommended path.
6. It provides a governed workstation inside the network with access to private endpoints and correct DNS, keeps data off personal devices, and applies Conditional Access and redirection controls.
7. Session host size and count, autoscale and scaling plans, depth-first load balancing, how many users share each host, and commitment discounts for the baseline.

</details>

## References

- [What is Azure Virtual Desktop?](https://learn.microsoft.com/azure/virtual-desktop/overview)
- [Azure Virtual Desktop service architecture and resilience](https://learn.microsoft.com/azure/virtual-desktop/service-architecture-resilience)
- [Host pool load balancing algorithms](https://learn.microsoft.com/azure/virtual-desktop/host-pool-load-balancing)
- [Autoscale scaling plans](https://learn.microsoft.com/azure/virtual-desktop/autoscale-scaling-plan)
- [FSLogix profile containers with Azure Files](https://learn.microsoft.com/azure/virtual-desktop/fslogix-profile-containers)
- [What's new in Azure Virtual Desktop?](https://learn.microsoft.com/azure/virtual-desktop/whats-new)
- [What is Windows 365?](https://learn.microsoft.com/windows-365/overview)
- [Windows 365 Enterprise overview](https://learn.microsoft.com/windows-365/enterprise/overview)
- [Microsoft Dev Box and Windows 365 announcement](https://learn.microsoft.com/azure/dev-box/dev-box-windows-365-announcement)
- [Azure Virtual Desktop landing zone accelerator](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/azure-virtual-desktop/)
