# Images and VM Image Builder

> Topic: 06 · Compute · Tracks: E ○ A ○ C – · Status checked: 2026-10

## Why it matters

Every VM, scale set instance and many AKS and Batch nodes start from an image. If that image is a marketplace default plus a long start-up script, provisioning is slow and fragile, and every instance may differ slightly. A versioned "golden" image built by a pipeline gives fast, identical, already-hardened instances. For GPU workloads the case is stronger: drivers, CUDA libraries and multi-gigabyte frameworks belong in the image, not in a boot script.

## Key concepts

### Image sources

| Source | What it is |
|---|---|
| **Azure Marketplace images** | Published by Microsoft and partners: Ubuntu, RHEL, SLES, Debian, Azure Linux, Windows Server, Windows 11, SQL Server, plus third-party appliances |
| **Specialised Microsoft images** | Data Science Virtual Machine, HPC images with InfiniBand drivers, GPU-optimised images |
| **Community gallery images** | Shared publicly by other publishers. Use with the same caution as any third-party software |
| **Your own images** | Built from the above and stored in an **Azure Compute Gallery** |

Marketplace images are identified by a URN: `Publisher:Offer:Sku:Version`, for example `Canonical:ubuntu-24_04-lts:server:latest`. Pin a version for reproducibility; `latest` changes underneath you.

Some marketplace images carry licence fees or require accepting terms before first use (`az vm image terms accept`).

### Generalised and specialised

| | **Generalised** | **Specialised** |
|---|---|---|
| Machine identity | Removed (Sysprep on Windows, `waagent -deprovision` on Linux) | Kept: host name, accounts, SIDs |
| Provisioning | The platform sets host name, admin user, keys | Boots exactly as captured |
| Use | Templates for many VMs and scale sets | Cloning or migrating a specific machine |

Golden images are generalised. Specialised images suit migration and quick duplication of a configured server.

### Azure Compute Gallery

The service for storing, versioning, replicating and sharing images.

```text
Gallery
└── Image definition          the "what": OS type, generation, architecture, security type, publisher/offer/SKU
    └── Image version          the "which": 1.0.0, 1.1.0 …  with replication regions and replica counts
```

| Capability | Detail |
|---|---|
| **Versioning** | Semantic versions. Deploy a specific version or `latest` |
| **Replication** | Copy each version to the regions where it will be used, with several replicas per region so large scale-outs are not throttled |
| **Zone-redundant storage** | For resilience of the image itself |
| **Lifecycle** | *Exclude from latest*, end-of-life dates, soft delete, and removal of old versions |
| **Sharing** | **RBAC** (within and across tenants), **direct shared gallery** (to specific subscriptions or tenants), **community gallery** (public) |
| **VM applications** | Versioned application packages delivered to VMs independently of the image |
| **Image definition properties** | Hyper-V generation, architecture (x64 or Arm64), **Trusted Launch / confidential** security type, accelerated networking, recommended sizes |

The older standalone **managed image** resource is legacy. Use gallery images.

An image definition's security type must match how it will be used: Trusted Launch images for Trusted Launch VMs, confidential-capable images for confidential VMs.

### Ways to build an image

| Method | Notes |
|---|---|
| **Capture a VM** | Configure by hand, generalise, capture. Fine for a one-off; not repeatable |
| **Azure VM Image Builder** | Managed service that builds from a template. Recommended on Azure |
| **Packer** | HashiCorp's open-source tool with an Azure builder. Good when you build for several platforms; Image Builder runs Packer underneath |
| **Bring a VHD** | Upload a prepared disk from elsewhere |

### Azure VM Image Builder

A managed service: you describe the image in a template, and it creates a temporary build VM, runs your steps, captures the result and distributes it.

| Template section | Purpose |
|---|---|
| **Source** | A marketplace image, an existing gallery image version, or a managed image |
| **Customise** | Ordered steps: **Shell** and **PowerShell** scripts (inline or from a URL), **File** copies, **WindowsUpdate**, **WindowsRestart** |
| **Validate** | Optional tests that must pass before distribution |
| **Distribute** | An Azure Compute Gallery version (with target regions and replica counts), a managed image, or a VHD |
| **Identity** | A **user-assigned managed identity** with rights to read the source and write to the gallery |
| **VM profile** | Build VM size, disk size, and an existing subnet for builds that need private access |
| **Optimise / error handling** | Boot optimisation; whether to keep the build VM on failure for debugging |

Other features:

- **Triggers:** rebuild automatically when the source image is updated, so monthly marketplace patches flow into your image.
- **Isolated builds** and private networking for regulated environments.
- **Build timeouts** and logs in a staging resource group.
- Fully definable in Bicep (`Microsoft.VirtualMachineImages/imageTemplates`) or through Azure Verified Modules, and runnable from GitHub Actions or Azure Pipelines.

A minimal template in Bicep:

```bicep
resource template 'Microsoft.VirtualMachineImages/imageTemplates@2024-02-01' = {
  name: 'it-ubuntu2404-base'
  location: location
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${builderIdentityId}': {} } }
  properties: {
    buildTimeoutInMinutes: 60
    vmProfile: { vmSize: 'Standard_D4as_v6', osDiskSizeGB: 64 }
    source: {
      type: 'PlatformImage'
      publisher: 'Canonical'
      offer: 'ubuntu-24_04-lts'
      sku: 'server'
      version: 'latest'
    }
    customize: [
      { type: 'Shell', name: 'patch',   inline: ['sudo apt-get update', 'sudo DEBIAN_FRONTEND=noninteractive apt-get -y upgrade'] }
      { type: 'Shell', name: 'harden',  scriptUri: 'https://<storage>/scripts/harden.sh' }
      { type: 'Shell', name: 'tooling', inline: ['sudo apt-get install -y jq unzip'] }
    ]
    validate: {
      inVMValidations: [ { type: 'Shell', name: 'smoke', inline: ['jq --version', 'test "$(sudo ufw status | head -1)" != ""'] } ]
    }
    distribute: [
      {
        type: 'SharedImage'
        runOutputName: 'ubuntu2404-base'
        galleryImageId: '${galleryImageDefinitionId}/versions/1.4.0'
        targetRegions: [
          { name: 'swedencentral', replicaCount: 2, storageAccountType: 'Standard_ZRS' }
          { name: 'westeurope',    replicaCount: 1 }
        ]
      }
    ]
  }
}
```

### What goes in the image, and what does not

| Bake into the image | Apply at deployment or boot |
|---|---|
| OS patches as of the build date | Environment-specific settings: endpoints, feature flags |
| Hardening baseline (CIS or organisational) | Secrets and certificates (fetched with managed identity from Key Vault) |
| Monitoring and security agents | Host name, domain join |
| Runtimes, language packs, system libraries | The application build, if it changes more often than the image |
| **GPU drivers, CUDA, container runtime, ML frameworks** | Model weights that change frequently (pull from storage or a registry) |
| Large, slow-to-install dependencies | |

Never bake secrets, keys or tokens into an image. Images are copied, shared and long-lived.

How much application to bake in is a trade-off. Fully baked images (application included) start fastest and are immutable, at the cost of an image build per release. Base images plus a deployment step build less often and start slower.

### Azure Linux

Microsoft's own Linux distribution (formerly CBL-Mariner): small, security-focused, RPM-based, and maintained with fast patching. It is the default or recommended host OS for **AKS nodes** and is available as a VM image and container base image. For container hosts it reduces attack surface and boot time compared with a general-purpose distribution.

### Images for GPU and AI workloads

| Option | Contents |
|---|---|
| **GPU driver VM extension** on a standard image | Installs NVIDIA or AMD drivers at provisioning. Simple, adds minutes to every boot, and depends on downloads succeeding |
| **Marketplace AI / HPC images** | Ubuntu- or AlmaLinux-based HPC images with InfiniBand and GPU drivers and MPI libraries preinstalled |
| **Data Science Virtual Machine** | Many ML frameworks and tools preinstalled. Good for individual exploration; large |
| **Your own GPU golden image** | Standard base, pinned driver and CUDA versions, container runtime with the NVIDIA toolkit, your frameworks. Best for production clusters |
| **Containers on a GPU-ready host** | The host image carries only drivers and the container runtime; frameworks live in the container image. The usual pattern on AKS |

Driver, CUDA and framework versions must match. Pinning them in an image and testing the image is far more reliable than installing whatever is current at boot.

## How it works on Azure

### A golden image pipeline

```text
Trigger: schedule (monthly), new marketplace version, or change to build scripts in Git
   │
   ▼
Azure VM Image Builder
   1. start a build VM from the source image
   2. patch, harden, install agents and runtimes
   3. validate (smoke tests, security scan, CIS check)
   4. generalise and capture
   5. publish gallery version x.y.z, replicate to regions
   │
   ▼
Test: deploy a canary instance from the new version; run acceptance tests
   │
   ▼
Roll out: scale sets with automatic OS image upgrades pick up the new version in health-gated batches
   │
   ▼
Retire: mark old versions end-of-life; remove after a retention period
```

Treat the image definition and build scripts as code in a repository, with review. The pipeline's identity needs Contributor on the staging resources and write access to the gallery, nothing broader.

### Keeping fleets current

| Fleet type | How new images reach it |
|---|---|
| Scale sets (image-based) | **Automatic OS image upgrade** to the latest gallery version, rolling with health checks |
| AKS node pools | **Node image upgrades** (automatic channels or planned), using Microsoft's node images or your own where supported |
| Long-lived individual VMs | Images do not update running VMs. Patch in place with Update Manager, or rebuild |
| Batch pools, virtual desktop host pools | Recreate or update pools and session hosts from the new version |

An image is a snapshot in time. A six-month-old image deploys six-month-old vulnerabilities, so rebuild at least monthly.

### Sharing across an organisation

- One or a few **central galleries** in a platform subscription.
- Workload subscriptions get **Reader** on the gallery, or direct sharing.
- Azure Policy can **restrict VM creation to approved gallery images** and block arbitrary marketplace images.
- Replicate versions to every region where workloads run, including disaster recovery regions. A region failover that depends on an image stored only in the failed region will not work.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Marketplace image plus script, or golden image | Golden image for anything deployed at scale, autoscaled, or security-sensitive |
| Image Builder or Packer | Image Builder for an Azure-only, managed approach. Packer when the same definitions build images for several platforms |
| How much to bake | More baking: faster start, more builds. Find the layer that changes rarely and stop there |
| One image or many | A small family: a hardened base per OS, then role images built from it (web, GPU, build agent) |
| Rebuild cadence | Monthly at minimum, plus on critical vulnerabilities |
| Replica count | More replicas for images used in large simultaneous scale-outs |
| Images or containers | If the workload can be containerised, a container image on a thin host is lighter and faster to iterate than VM images |

## Hands-on

```bash
RG=rg-image-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Explore marketplace images
az vm image list-publishers -l $LOC --query "[?contains(name,'Canonical') || contains(name,'MicrosoftWindowsServer') || contains(name,'microsoft-dsvm')].name" -o tsv
az vm image list --publisher Canonical --offer ubuntu-24_04-lts --all -l $LOC --query "[-3:].{Urn:urn, Arch:architecture}" -o table
az vm image show --urn Canonical:ubuntu-24_04-lts:server:latest -l $LOC --query "{gen:hyperVGeneration, features:features}"

# 2. A gallery and an image definition
az sig create -g $RG --gallery-name galimagelab -o none
az sig image-definition create -g $RG --gallery-name galimagelab --gallery-image-definition ubuntu2404-base \
  --publisher contoso --offer ubuntu --sku 2404-base --os-type Linux --os-state Generalized \
  --hyper-v-generation V2 --features SecurityType=TrustedLaunch -o none

# 3. Build a version the simple way: configure a VM, generalise, capture
az vm create -g $RG -n vm-build --image Ubuntu2404 --size Standard_D2as_v6 --public-ip-address "" \
  --security-type TrustedLaunch --admin-username azureuser --generate-ssh-keys -o none
az vm run-command invoke -g $RG -n vm-build --command-id RunShellScript \
  --scripts "echo 'built $(date -u +%F)' > /etc/image-info; waagent -deprovision+user -force" -o none
az vm deallocate -g $RG -n vm-build -o none
az vm generalize -g $RG -n vm-build -o none
az sig image-version create -g $RG --gallery-name galimagelab --gallery-image-definition ubuntu2404-base \
  --gallery-image-version 1.0.0 --virtual-machine $(az vm show -g $RG -n vm-build --query id -o tsv) \
  --target-regions $LOC=1 --replica-count 1 -o none

# 4. Deploy from the gallery
IMG=$(az sig image-definition show -g $RG --gallery-name galimagelab --gallery-image-definition ubuntu2404-base --query id -o tsv)
az vm create -g $RG -n vm-from-image --image $IMG --size Standard_D2as_v6 --public-ip-address "" \
  --security-type TrustedLaunch --admin-username azureuser --generate-ssh-keys -o none
az vm run-command invoke -g $RG -n vm-from-image --command-id RunShellScript --scripts "cat /etc/image-info; hostname" \
  --query "value[0].message" -o tsv

az sig image-version list -g $RG --gallery-name galimagelab --gallery-image-definition ubuntu2404-base \
  --query "[].{Version:name, Regions:publishingProfile.targetRegions[].name}" -o table

az group delete -n $RG --yes --no-wait
```

Then do it properly: follow the *Create a Linux image and distribute it to an Azure Compute Gallery* quickstart for VM Image Builder, with a user-assigned identity and a Bicep template, and build version 1.1.0 with a customiser script and a validation step. Compare how long a VM from your image takes to become ready against one that installs the same packages with cloud-init.

## Security, cost and reliability notes

- **Security:** an image is a supply-chain artefact. Build from trusted sources, scan it, sign or attest where supported, and control who can publish to the gallery.
- **Security:** no secrets in images. Check build scripts and shell history are cleaned.
- **Security:** restrict allowed images with Azure Policy so workloads cannot start from arbitrary marketplace or community images.
- **Security:** stale images are vulnerable images. Automate rebuilds and retire old versions.
- **Cost:** image versions cost storage per replica per region. Prune old versions. Build VMs run only for the build.
- **Reliability:** replicate images to disaster recovery regions. Keep the previous version available for rollback.
- **Reliability:** validate before distributing. A broken image reaching an auto-upgrading fleet is a fleet-wide incident; health-gated rolling upgrades are the safety net.

## Common mistakes

- Hand-built images nobody can reproduce.
- `latest` in production templates with no pinning or testing.
- Images rebuilt once a year.
- Secrets, keys or internal credentials baked in.
- GPU drivers installed at boot on every node, with unpinned versions.
- An image stored in one region only.
- Mismatch between the image definition's generation or security type and the VM being created.
- Enormous everything-included images when a small base plus containers would do.
- Old versions never deleted.
- Using the legacy managed image resource for new work.

## Check yourself

1. What is the difference between a generalised and a specialised image?
2. Describe the hierarchy in an Azure Compute Gallery.
3. What are the main sections of a VM Image Builder template?
4. Give three things to bake into an image and two to leave out.
5. Why is a golden image better than a long boot script for autoscaled fleets?
6. How do new image versions reach a running scale set?
7. Why should images be replicated to more than one region?
8. What are the options for getting GPU drivers onto a VM, and which is best for production?

<details>
<summary>Answers</summary>

1. A generalised image has machine identity removed and is provisioned with new host name and credentials, for use as a template. A specialised image keeps its identity and boots as captured.
2. A gallery contains image definitions; each definition contains image versions, which are replicated to target regions.
3. Source, customise, validate, distribute, plus the identity and build VM profile.
4. Bake in: patches, hardening, agents, runtimes, GPU drivers, slow-to-install dependencies. Leave out: secrets and certificates, environment-specific configuration, frequently changing application builds or model weights.
5. Instances start faster and identically, with no dependence on package downloads at boot, and are already patched and hardened.
6. Through automatic OS image upgrades (or a model update with a rolling upgrade policy), in health-gated batches.
7. So deployments, scale-out and failover in other regions do not depend on a region that may be unavailable, and to reduce deployment latency.
8. A driver extension at provisioning, a marketplace HPC or AI image, the Data Science VM, or a custom golden image with pinned driver and CUDA versions. The custom golden image (or containers on a driver-ready host) is best for production.

</details>

## References

- [Store and share images in an Azure Compute Gallery](https://learn.microsoft.com/azure/virtual-machines/azure-compute-gallery)
- [Azure VM Image Builder overview](https://learn.microsoft.com/azure/virtual-machines/image-builder-overview)
- [Create an Azure Image Builder Bicep or ARM template](https://learn.microsoft.com/azure/virtual-machines/linux/image-builder-json)
- [Find and use Azure Marketplace VM images](https://learn.microsoft.com/azure/virtual-machines/linux/cli-ps-findimage)
- [Remove machine-specific information by generalizing a VM](https://learn.microsoft.com/azure/virtual-machines/generalize)
- [Automatic OS image upgrades for scale sets](https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-automatic-upgrade)
- [VM applications overview](https://learn.microsoft.com/azure/virtual-machines/vm-applications)
- [What is Azure Linux?](https://learn.microsoft.com/azure/azure-linux/intro-azure-linux)
- [NVIDIA GPU Driver Extension for Linux](https://learn.microsoft.com/azure/virtual-machines/extensions/hpccompute-gpu-linux)
- [Packer: Azure builder](https://developer.hashicorp.com/packer/integrations/hashicorp/azure)
