# Virtual Machines

> Topic: 06 · Compute · Tracks: E ○ A ● C ○ · Status checked: 2026-10

## Why it matters

The virtual machine is the base unit of cloud compute. AKS nodes, scale sets, Batch pools, virtual desktops and GPU clusters are all VMs underneath, so sizing, pricing and availability concepts learned here carry through the rest of the platform. Even if you never run an application directly on a VM, you will choose VM sizes for node pools and GPU workloads, and you will meet lifted-and-shifted VMs in most client estates.

## Key concepts

### What you get, and what you take on

A VM gives you a virtualised server: CPUs, memory, disks and network interfaces, with an operating system you control. It is IaaS, so per the shared responsibility model you own the guest OS and everything above it: patching, hardening, antimalware, backup, monitoring and high availability.

That operational load is the main reason to prefer a managed platform when one fits. Reasons to choose a VM anyway:

- Software that needs OS-level access, a specific kernel, drivers or licensing tied to a machine.
- Lift-and-shift migration with minimal change.
- Specialised hardware: GPUs, high memory, HPC interconnect, local NVMe.
- Third-party appliances and commercial software certified only on VMs.
- Full control over the stack for regulatory or performance reasons.

### Resources that make up a VM

| Resource | Notes |
|---|---|
| Virtual machine | The compute allocation and configuration |
| OS disk | Managed disk holding the operating system |
| Data disks | Optional managed disks for application data |
| Temporary disk | Local, ephemeral storage on some sizes. **Lost on deallocation, redeployment and host maintenance** |
| Network interface(s) | In a subnet, with a private IP and optional public IP |
| Network security group | On the subnet or NIC |
| Managed identity | For keyless access to other services |
| Extensions | Agents and configuration scripts |

### Reading a size name

```text
Standard_D8ads_v6
         │││││  └─ version (hardware generation)
         ││││└──── s = supports Premium SSD storage
         │││└───── d = has a local temporary disk
         ││└────── a = AMD processor      (p = Arm, no letter = Intel)
         │└─────── 8 = vCPUs
         └──────── D = family (general purpose)

Standard_NC24ads_A10_v4      NC family, 24 vCPUs, AMD, local disk, premium storage, NVIDIA A10 GPU, version 4
Standard_E16-4ds_v6          16-vCPU memory size constrained to 4 active vCPUs (for per-core licensing)
```

| Letter | Meaning |
|---|---|
| `a` | AMD processor |
| `p` | Arm processor (Ampere or Microsoft Cobalt) |
| `d` | Local temporary disk present |
| `s` | Premium storage capable |
| `l` | Low memory per vCPU |
| `m` | Memory-intensive within the family |
| `b` | Higher remote storage (block) performance |
| `n` | NVMe disk interface (assumed on newest generations) |
| `i` | Isolated: the VM has the physical host to itself |
| `t` | Tiny memory |
| `C` | Confidential |
| `e` | Intel TDX confidential (in `DCe`/`ECe` series) |

### Size families

| Family | Purpose | Typical uses |
|---|---|---|
| **B** (burstable) | Low baseline CPU with credits for bursts | Dev/test, small web servers, jump boxes |
| **D** | General purpose, balanced CPU and memory (about 4 GiB per vCPU) | Most application servers, AKS system and general node pools |
| **E** | Memory optimised (about 8 GiB per vCPU) | Databases, caches, in-memory analytics |
| **F** | Compute optimised (about 2 GiB per vCPU, higher clock speed) | Batch processing, web front ends, gaming |
| **L** | Storage optimised, large local NVMe | NoSQL databases, data warehousing, large caches |
| **M** | Very large memory, up to multiple terabytes | SAP HANA, large in-memory databases |
| **N** | GPU: **NC** (compute and inference), **ND** (large-scale training and inference), **NV** (visualisation and VDI) | AI, rendering, simulation (file 07) |
| **H** | High performance computing: **HB**, **HC**, **HX**, with InfiniBand | CFD, weather, finite element analysis (file 09) |
| **DC / EC** | Confidential computing | Sensitive data processing (file 06) |
| **A** | Entry-level, older | Avoid for new work |

**Versions.** A higher version number means newer processors and platform features. Current generations use **NVMe** disk interfaces and the **Azure Boost** offload platform for better storage and network performance, and include **Arm-based** sizes on Microsoft's Cobalt processors with strong price-performance for scale-out Linux workloads. Older series are retired on a published schedule and some face capacity restrictions before retirement, so prefer current generations and plan migrations off old ones.

Moving to a newer generation is not always a drop-in resize: NVMe-only sizes need OS support and Generation 2 images, and the presence or absence of a local temporary disk (`d`) changes the VM's disk layout.

### Generation 1 and Generation 2

| | Generation 1 | Generation 2 |
|---|---|---|
| Firmware | BIOS | UEFI |
| Trusted Launch | No | Yes |
| Confidential VMs, newest sizes, NVMe | No | Required |
| OS disk size limit | 2 TiB | Larger |

Use Generation 2 for all new VMs.

### Trusted Launch

The default security type for new Generation 2 VMs:

- **Secure Boot:** only signed boot components load.
- **vTPM:** a virtual Trusted Platform Module for measured boot and key protection.
- **Boot integrity monitoring:** attestation through Defender for Cloud.

It protects against boot kits and rootkits at no extra cost. Keep it on.

### Provisioning and configuration

| Mechanism | Use |
|---|---|
| **Image** | Marketplace, or your own from an Azure Compute Gallery (file 05) |
| **cloud-init** (Linux) | Declarative first-boot configuration: packages, files, users, commands |
| **Custom data / user data** | A script or configuration passed at creation. User data can be updated later and read from the metadata service |
| **Extensions** | Post-deployment agents and tasks: Azure Monitor Agent, Custom Script, Entra login, GPU drivers, antimalware, Key Vault certificates |
| **Run Command** | Execute a script on a running VM through the control plane |
| **Machine configuration** (Azure Policy) | Audit and enforce in-guest settings |
| **VM applications** | Versioned application packages from a gallery |

Prefer baking software into an **image** and keeping first-boot configuration small. It gives faster, more reliable scale-out than long install scripts.

### Instance Metadata Service

A REST endpoint at `169.254.169.254`, reachable only from inside the VM. It returns the VM's own metadata (size, region, zone, tags), **scheduled events**, attested data, and **managed identity tokens**.

```bash
curl -s -H Metadata:true "http://169.254.169.254/metadata/instance?api-version=2021-02-01" | jq .compute.vmSize
curl -s -H Metadata:true "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01"
```

### Power states and billing

| State | Compute billed? | Notes |
|---|:-:|---|
| Running | Yes | |
| **Stopped** (shut down from inside the OS) | **Yes** | The allocation is still held |
| **Stopped (deallocated)** | No | Hardware released. Dynamic public IP and temporary disk contents are lost. Disks still billed |
| **Hibernated** | No | Memory saved to the OS disk; resumes where it left off. Supported on selected sizes and OS versions |

Shutting down inside the guest does not stop the charge. Deallocate with the portal, `az vm deallocate`, or auto-shutdown.

### Maintenance and updates

| Mechanism | What it does |
|---|---|
| **Live migration / memory-preserving updates** | Most host maintenance pauses the VM for seconds at most |
| **Scheduled Events** | A metadata-service notification, typically minutes ahead, of reboots, redeployments and Spot evictions, so the application can drain |
| **Maintenance configurations** | Control when platform updates apply to dedicated hosts, isolated VMs and some other resources |
| **Azure Update Manager** | Assess and install guest OS updates on a schedule across Azure and Arc-enabled machines |
| **Automatic VM guest patching** | Platform-orchestrated installation of critical and security patches |
| **Hotpatching** | Security updates without a reboot, for supported Windows Server editions and images |

### Pricing

| Component | Notes |
|---|---|
| **Compute** | Per second, by size. Includes the OS licence for Windows and paid Linux distributions unless you bring your own |
| **Disks** | By type and size, billed whether the VM runs or not |
| **Networking** | Public IPs, outbound data, load balancers |
| **Licences** | Windows Server, SQL Server, RHEL, SLES |

Ways to pay less (file 03 covers the first four in detail):

| Lever | Saving | Commitment |
|---|---|---|
| **Reservations** | Up to about 72% | 1 or 3 years, a VM family in a region |
| **Savings plan for compute** | Up to about 65% | 1 or 3 years, an hourly spend across compute services |
| **Spot** | Up to about 90% | None; can be evicted |
| **Azure Hybrid Benefit** | Removes the Windows or SQL licence cost | Existing licences with Software Assurance or subscriptions |
| **Dev/Test pricing** | No Windows licence charge | Dev/Test subscription offer |
| **Right-sizing** | Often the largest saving | None |
| **Auto-shutdown / start-stop schedules** | Proportional to idle time | None |
| **Arm and AMD sizes** | Lower price for similar performance | Application compatibility |

### Other features worth knowing

| Feature | Purpose |
|---|---|
| **Ephemeral OS disk** | OS disk on local storage: faster reimage, no storage cost, state lost on deallocation. Ideal for stateless scale sets and AKS nodes |
| **Accelerated networking** | SR-IOV for lower latency and higher throughput; default on modern sizes |
| **Proximity placement groups** | Place VMs physically close for low latency. Reduces placement flexibility; can conflict with zone spreading |
| **Boot diagnostics and serial console** | See boot output and get console access when the network path is broken |
| **VM restore points** | Application- or crash-consistent point-in-time copies of all disks |
| **Azure Backup, Site Recovery** | Backup and disaster recovery (topic 28) |
| **Azure Linux** | Microsoft's own Linux distribution, used heavily for AKS nodes |

## How it works on Azure

### Choosing a size

1. **Family** from the workload's shape: balanced (D), memory-heavy (E), CPU-heavy (F), GPU (N).
2. **Processor**: try AMD (`a`) and Arm (`p`) variants; they are usually cheaper for equivalent work if the software supports them.
3. **Current generation** available in your region and zones.
4. **Size** from measured need. Start modestly; resizing is a restart.
5. **Check the limits that bite**: maximum data disks, **uncached disk IOPS and throughput** (the VM caps what its disks can deliver), network bandwidth, NIC count, local disk.
6. **Quota**: vCPU quota per family per region. Request increases ahead of need.
7. **Price** with the pricing calculator, including disks and licences.

```bash
# Sizes of a family in a region, with zones and restrictions for your subscription
az vm list-skus -l swedencentral --resource-type virtualMachines --size Standard_D \
  --query "[?contains(name,'_v6')].{Size:name, vCPU:capabilities[?name=='vCPUs'].value|[0], MemGB:capabilities[?name=='MemoryGB'].value|[0], Zones:join(',',locationInfo[0].zones), Restricted:restrictions[0].reasonCode}" -o table
```

**Right-size from data.** Azure Monitor metrics and Azure Advisor recommendations show sustained under-use. Lifted-and-shifted VMs are frequently two sizes too large.

### A secure baseline VM

- No public IP; access through Bastion with Entra ID login (topic 05, file 14).
- Generation 2 with Trusted Launch.
- Managed identity; no credentials on disk.
- Encryption at host enabled (file 04).
- Azure Monitor Agent with a data collection rule; boot diagnostics on.
- Update Manager schedule or automatic guest patching.
- Defender for Servers.
- Backup policy for stateful VMs.
- Deployed from a hardened image, through IaC.
- In a private subnet with an explicit, controlled egress path.

Azure Policy and the landing zone baseline enforce most of this automatically.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| VM or a managed platform | Managed platform unless one of the reasons at the top of this file applies. See file 12 |
| Fewer large or more small | More small instances behind a load balancer give availability and elasticity. Large instances suit software that only scales up |
| Intel, AMD or Arm | Test AMD and Arm for price-performance. Arm needs Arm builds of everything, including container images |
| With or without a temporary disk | Without (`d`-less) sizes are cheaper; choose `d` when you need local scratch, swap, or an ephemeral OS disk |
| Burstable (B) | Good for spiky low-average workloads. Sustained load exhausts credits and throttles the VM |
| Pets or cattle | Treat VMs as replaceable: image-based, stateless where possible, recreated not repaired |
| Windows or Linux | Linux is cheaper (no licence) and is the norm for containers and AI tooling |

## Hands-on

```bash
RG=rg-vm-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Explore sizes and your quota
az vm list-usage -l $LOC --query "[?contains(name.value,'standardD') || name.value=='cores'].{Name:name.localizedValue, Used:currentValue, Limit:limit}" -o table
az vm image list --publisher Canonical --offer ubuntu-24_04-lts --all -l $LOC --query "[-3:].urn" -o tsv

# 2. cloud-init: install a web server on first boot
cat > cloud-init.yaml <<'EOF'
#cloud-config
package_update: true
packages: [nginx, jq]
write_files:
  - path: /var/www/html/index.html
    content: "<h1>Provisioned by cloud-init</h1>"
EOF

# 3. Create a VM with no public IP, a system-assigned identity and Trusted Launch
az vm create -g $RG -n vm-lab-01 --image Ubuntu2404 --size Standard_D2as_v6 \
  --public-ip-address "" --assign-identity --security-type TrustedLaunch \
  --enable-secure-boot true --enable-vtpm true --encryption-at-host true \
  --custom-data cloud-init.yaml --admin-username azureuser --generate-ssh-keys \
  --tags env=lab owner=me -o none
# If encryption at host fails: az feature register --namespace Microsoft.Compute --name EncryptionAtHost

# 4. Inspect from inside without logging in
az vm run-command invoke -g $RG -n vm-lab-01 --command-id RunShellScript --scripts '
curl -s -H Metadata:true "http://169.254.169.254/metadata/instance/compute?api-version=2021-02-01" | jq "{vmSize, location, zone, securityProfile}"
curl -s localhost | head -1
lsblk' --query "value[0].message" -o tsv

# 5. Power states and billing
az vm stop       -g $RG -n vm-lab-01 -o none; az vm get-instance-view -g $RG -n vm-lab-01 --query "instanceView.statuses[1].displayStatus"
az vm deallocate -g $RG -n vm-lab-01 -o none; az vm get-instance-view -g $RG -n vm-lab-01 --query "instanceView.statuses[1].displayStatus"

# 6. Resize while deallocated, then start
az vm list-vm-resize-options -g $RG -n vm-lab-01 --query "[?contains(name,'D4')].name" -o tsv | head
az vm resize -g $RG -n vm-lab-01 --size Standard_D4as_v6 -o none
az vm start  -g $RG -n vm-lab-01 -o none

az group delete -n $RG --yes --no-wait
```

A new subnet has no outbound path, so the `packages` step may fail unless the subnet has a NAT gateway; the default VNet created by `az vm create` behaves according to the API version in use. If nginx is missing, that is the default-outbound change from topic 05 in action. Substitute an available size if `D2as_v6` is not offered in your region.

Exercises:

1. Decode: `Standard_E32pds_v6`, `Standard_F16s_v2`, `Standard_L8as_v3`, `Standard_NC48ads_H100_v5`.
2. Use the pricing calculator to compare one `D4as_v6` pay-as-you-go, with a 3-year reservation, and as Spot, for a month.

## Security, cost and reliability notes

- **Security:** unpatched internet-facing VMs are a leading cause of compromise. No public IPs, automated patching, Defender for Servers.
- **Security:** never store credentials in custom data or scripts; they are readable from the metadata service and deployment history. Use managed identity and Key Vault.
- **Cost:** VMs bill when *stopped* but not *deallocated*; disks and reserved public IPs always bill. Idle and oversized VMs are the most common waste in a subscription.
- **Reliability:** a single VM is a single point of failure whatever its SLA. Use zones and multiple instances (file 02).
- **Reliability:** the temporary disk is not durable. Never put anything on it you cannot lose.

## Common mistakes

- Choosing a VM where a managed platform would do, and inheriting patching and availability work.
- Stopping instead of deallocating and still paying.
- Data on the temporary disk.
- Public IPs with open management ports.
- Sizing by the on-premises server's specification.
- Ignoring the VM's own disk throughput cap, then blaming the disk.
- Old-generation sizes for new deployments.
- Hand-configured servers nobody can rebuild.
- Long install scripts at boot instead of a prepared image.
- Forgetting vCPU quota until a scale-out fails.

## Check yourself

1. Decode `Standard_D8ads_v6`.
2. What are you responsible for on a VM that you are not on App Service?
3. Which power state stops compute billing?
4. What does Trusted Launch provide?
5. What is the Instance Metadata Service used for?
6. What happens to the temporary disk on deallocation?
7. Name four ways to reduce VM cost.
8. Why prefer an image over a long first-boot script?

<details>
<summary>Answers</summary>

1. General purpose (D), 8 vCPUs, AMD processor, local temporary disk, Premium storage capable, version 6.
2. Guest OS patching and hardening, runtime, antimalware, in-guest configuration, and building high availability.
3. Stopped (deallocated), or hibernated.
4. Secure Boot, a virtual TPM and boot integrity monitoring.
5. Reading the VM's own metadata, scheduled events (maintenance and eviction notices) and managed identity tokens from inside the VM.
6. Its contents are lost.
7. Any four of: reservations, savings plans, Spot, Azure Hybrid Benefit, right-sizing, auto-shutdown, AMD or Arm sizes, Dev/Test pricing.
8. Faster and more reliable provisioning and scale-out, with fewer external dependencies at boot.

</details>

## References

- [Virtual machines in Azure](https://learn.microsoft.com/azure/virtual-machines/overview)
- [Sizes for virtual machines in Azure](https://learn.microsoft.com/azure/virtual-machines/sizes/overview)
- [Azure VM sizes naming conventions](https://learn.microsoft.com/azure/virtual-machines/vm-naming-conventions)
- [Trusted Launch for Azure virtual machines](https://learn.microsoft.com/azure/virtual-machines/trusted-launch)
- [States and billing status of Azure VMs](https://learn.microsoft.com/azure/virtual-machines/states-billing)
- [cloud-init support for virtual machines](https://learn.microsoft.com/azure/virtual-machines/linux/using-cloud-init)
- [Azure Instance Metadata Service](https://learn.microsoft.com/azure/virtual-machines/instance-metadata-service)
- [Scheduled Events](https://learn.microsoft.com/azure/virtual-machines/linux/scheduled-events)
- [Azure Update Manager](https://learn.microsoft.com/azure/update-manager/overview)
- [Retired and retiring VM sizes](https://learn.microsoft.com/azure/virtual-machines/sizes/retirement/retired-sizes-list)
