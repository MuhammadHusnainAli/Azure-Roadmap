# Azure Files and File Sync

## Why it matters

Many applications, and most office file servers, expect a shared file system with folders, locks and Windows permissions. Azure Files provides fully managed SMB and NFS shares, and Azure File Sync turns on-premises Windows file servers into caches of those shares. Together they are a common first step in moving file servers to Azure, hosting user profiles for virtual desktops, and giving lift-and-shift applications shared storage without running file server VMs.

## Key concepts

### Protocols

| Protocol | Clients | Notes |
|---|---|---|
| **SMB 3.x** | Windows, macOS, Linux | Identity-based auth, Windows ACLs, encryption in transit; port 445 must be reachable |
| **NFS 4.1** | Linux | Premium (SSD) shares only; network-based access control (no user auth); private networking required |
| **REST (FileREST)** | Any HTTP client, SDKs | Programmatic access |

A share uses one protocol: SMB or NFS, not both.

### Tiers and billing models

| Media tier | Performance | Use |
|---|---|---|
| **SSD (premium)** | Low latency, high IOPS | Databases on shares, virtual desktop profiles, latency-sensitive apps, NFS |
| **HDD (standard)** | Higher latency | General file shares, archives, team shares |

Billing models:

| Model | How it works |
|---|---|
| **Provisioned v2** | You provision storage, IOPS and throughput independently (now the recommended model for new shares, with SSD and HDD options; check regional availability) |
| **Provisioned v1** | Premium SSD shares: provision size; IOPS and throughput scale with size |
| **Pay-as-you-go** | Standard HDD: pay for used capacity and transactions, with **transaction optimised, hot and cool** access tiers |

Provisioned models make costs predictable and avoid transaction charges that surprise pay-as-you-go users.

### Identity-based access for SMB

| Source | Clients | Fit |
|---|---|---|
| **On-premises AD DS** | Domain-joined machines with line of sight to DCs | Hybrid estates with existing AD |
| **Microsoft Entra Domain Services** | VMs joined to the managed domain | Cloud-only lift-and-shift needing Kerberos and NTLM |
| **Microsoft Entra Kerberos** | Entra-joined or hybrid-joined devices, hybrid identities (and cloud-only identities in newer releases; check current status) | Virtual desktops (FSLogix), modern devices without DC line of sight |
| Storage account key | Any | Admin and emergency use only |

Two levels of permission apply:

1. **Share-level**: Azure RBAC roles (`Storage File Data SMB Share Reader`, `Contributor`, `Elevated Contributor`), or a default share-level permission for all authenticated identities.
2. **Directory and file level**: Windows ACLs (NTFS permissions), set with Windows tools or `icacls`.

### Protection

| Feature | Detail |
|---|---|
| Share snapshots | Read-only point-in-time copies; exposed to users as "Previous versions" in Windows |
| Soft delete | Recover deleted shares |
| Azure Backup | Snapshot-based and vaulted backup policies for shares |
| Redundancy | LRS, ZRS (premium and standard), GRS and GZRS for standard (with size limits on some) |

### Azure File Sync

```text
On-premises Windows Server(s) with File Sync agent
   ├─ server endpoint  D:\Shares\Finance  (cache, cloud tiering on)
   │           ▲ sync (multi-master, changes detected via USN journal)
   ▼           │
Storage Sync Service ─ sync group "Finance"
   └─ cloud endpoint: Azure file share "finance"   ← authoritative copy, backed up in Azure
```

| Capability | Detail |
|---|---|
| **Cloud tiering** | Keeps hot files locally; cold files become pointers, recalled on access. Policy by free space percentage and date |
| **Multi-site** | Several servers in different offices sync the same share (last writer wins on conflicts, with conflict copies) |
| **Fast disaster recovery** | Rebuild a server and restore namespace quickly; data recalls on demand |
| **Migration path** | Sync, then point users to the Azure share or keep the server as cache |
| **Managed identity** | Sync service and servers can use managed identities instead of keys (check current status for Arc-enabled servers) |

## How it works on Azure

### Networking

- SMB uses TCP 445, which many ISPs and corporate networks block outbound. Access over VPN or ExpressRoute with a **private endpoint** is the normal enterprise pattern.
- NFS shares require private endpoints or service endpoints; no public access.
- Private DNS zone `privatelink.file.core.windows.net`.

### Common scenarios

| Scenario | Design |
|---|---|
| Replace a file server | Azure Files (SMB, AD DS or Entra Kerberos auth) + File Sync as on-site cache during transition |
| Azure Virtual Desktop FSLogix profiles | Premium SSD share, Entra Kerberos or AD DS, ZRS |
| Lift-and-shift app needing a shared folder | SMB share with private endpoint; managed identity or AD for the app |
| Linux apps needing shared POSIX storage | NFS on SSD; or Azure NetApp Files for higher performance |
| Container shared volumes | Azure Files CSI driver on AKS (topic 07 file 13) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Azure Files or Azure NetApp Files | Azure Files for most file shares; NetApp Files for very low latency, high throughput, large enterprise NAS features (file 07) |
| SSD or HDD | SSD for profiles, databases, NFS and interactive apps; HDD for general team shares and archives |
| Billing model | Provisioned v2 for predictable spend on new shares; pay-as-you-go only for low, well-understood transaction volumes |
| Identity source | Entra Kerberos for modern, cloud-joined devices; AD DS for classic domain environments |
| File Sync or direct access | File Sync when users need local performance or offline resilience at branch offices; direct access where network latency is acceptable |
| Redundancy | ZRS for production; GZRS or backup to another region for DR |

## Hands-on

```bash
RG=rg-files-lab; LOC=swedencentral; SA=stfiles$RANDOM
az group create -n $RG -l $LOC -o none

# 1. Standard account and a hot-tier SMB share
az storage account create -g $RG -n $SA -l $LOC --kind StorageV2 --sku Standard_ZRS -o none
az storage share-rm create -g $RG --storage-account $SA -n team --access-tier Hot --quota 100 -o none

# 2. Snapshot and soft delete
az storage account file-service-properties update -g $RG -n $SA --enable-delete-retention true --delete-retention-days 14 -o none
az storage share-rm snapshot -g $RG --storage-account $SA -n team -o none
az storage share-rm list -g $RG --storage-account $SA --include-snapshot -o table

# 3. Mount from Linux over SMB (needs outbound 445; uses the key here for the lab only)
KEY=$(az storage account keys list -g $RG -n $SA --query "[0].value" -o tsv)
sudo mkdir -p /mnt/team
sudo mount -t cifs //$SA.file.core.windows.net/team /mnt/team \
  -o vers=3.1.1,username=$SA,password=$KEY,serverino,nosharesock,actimeo=30,mfsymlinks
echo hello | sudo tee /mnt/team/hello.txt; ls -l /mnt/team

az group delete -n $RG --yes --no-wait
```

For a production design, repeat with a private endpoint and identity-based access instead of the key, and walk through the File Sync quickstart on a test Windows Server VM.

## Security, cost and reliability notes

- **Security:** use identity-based access and disable key-based access where possible; require SMB 3.1.1 with AES-256-GCM encryption via the share's security profile.
- **Security:** private endpoints for enterprise shares; never open SMB to the internet.
- **Cost:** pay-as-you-go transaction charges can exceed capacity charges for chatty workloads (sync, antivirus scans, profiles). Provisioned billing avoids this.
- **Cost:** File Sync cloud tiering reduces on-premises storage but each recall is a transaction and egress cost.
- **Reliability:** back up shares with Azure Backup; snapshots alone live in the same account.

## Common mistakes

- Mounting with the account key everywhere and never moving to identity-based access.
- Trying to reach SMB over the internet from a network that blocks port 445.
- Choosing HDD for FSLogix profiles, then poor login performance.
- Expecting NFS shares to enforce per-user permissions like SMB.
- Pay-as-you-go shares with heavy transaction loads and surprise bills.
- Treating File Sync as a backup.

## Check yourself

1. Which protocols do Azure Files support and what are their key differences?
2. What are the two levels of permissions on an identity-based SMB share?
3. When would you use Entra Kerberos rather than AD DS authentication?
4. What does cloud tiering do in Azure File Sync?
5. Why is port 445 a common obstacle, and what is the usual solution?
6. Why might a provisioned billing model be cheaper than pay-as-you-go?

<details>
<summary>Answers</summary>

1. SMB 3.x (Windows, macOS, Linux; identity-based auth and ACLs) and NFS 4.1 (Linux, SSD only, network-level access control, private networking required), plus REST.
2. Share-level permissions through Azure RBAC, and directory and file permissions through Windows ACLs.
3. For Entra-joined or hybrid-joined clients (such as virtual desktops) without line of sight to domain controllers.
4. It keeps frequently used files locally and replaces cold files with pointers that are recalled from Azure on access.
5. Many ISPs and networks block outbound TCP 445. Use VPN or ExpressRoute with a private endpoint.
6. It has no per-transaction charges, so transaction-heavy workloads cost less and spend is predictable.

</details>

## References

- [Azure Files introduction](https://learn.microsoft.com/azure/storage/files/storage-files-introduction)
- [Planning for an Azure Files deployment](https://learn.microsoft.com/azure/storage/files/storage-files-planning)
- [Understanding Azure Files billing](https://learn.microsoft.com/azure/storage/files/understanding-billing)
- [Identity-based authentication for SMB](https://learn.microsoft.com/azure/storage/files/storage-files-active-directory-overview)
- [Entra Kerberos for Azure Files](https://learn.microsoft.com/azure/storage/files/storage-files-identity-auth-hybrid-identities-enable)
- [NFS file shares](https://learn.microsoft.com/azure/storage/files/files-nfs-protocol)
- [Azure File Sync planning](https://learn.microsoft.com/azure/storage/file-sync/file-sync-planning)
- [Cloud tiering overview](https://learn.microsoft.com/azure/storage/file-sync/file-sync-cloud-tiering-overview)
