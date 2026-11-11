# Encryption

## Why it matters

Encryption is the control auditors ask about first and the one most often misunderstood. Azure already encrypts data at rest by default and protects traffic between datacentres, so "turn on encryption" is rarely the real question. The real questions are: who controls the keys, what happens if a key is lost or revoked, is the data protected while it's being processed, which protocols are clients allowed to use, and does a second layer of encryption reduce any real risk? Answering these precisely lets architects meet regulatory requirements without adding cost and operational risk for no benefit.

## Key concepts

### Three states of data

| State | Azure default | Additional options |
|---|---|---|
| **At rest** | Service-side encryption (AES-256) with Microsoft-managed keys for Storage, managed disks, SQL (TDE), Cosmos DB and most PaaS | Customer-managed keys (CMK), infrastructure (double) encryption, encryption at host, client-side encryption |
| **In transit** | TLS for Azure service endpoints (TLS 1.2+ required; most services completed the transition by 31 August 2025); **MACsec** (IEEE 802.1AE) on links between Microsoft datacentres outside Microsoft's physical control | Enforce minimum TLS version, private endpoints, VPN/ExpressRoute with MACsec or IPsec |
| **In use** | Not encrypted in memory by default | **Confidential computing**: confidential VMs (AMD SEV-SNP, Intel TDX), confidential containers, Intel SGX enclaves, confidential GPUs |

### Key management models (server-side)

| Model | Who controls the key | Effort | Use |
|---|---|---|---|
| **Microsoft-managed keys** (default) | Microsoft | None | Most workloads |
| **Customer-managed keys (CMK)** | You, in Key Vault or Managed HSM | Key lifecycle, access, availability | Regulatory requirement for key control, ability to revoke access (crypto-shredding) |
| **Customer-provided keys** | You supply the key per request (for example Blob Storage) | High | Rare, specific needs |
| **Client-side encryption** | Your app encrypts before sending | Highest; service can't index or process plaintext | Data the service must never see in clear |

CMK support varies by service; check the "services supporting CMK" list. Use **versionless key URIs** so services pick up rotated key versions automatically (file 03).

### Double encryption

| Layer | Example |
|---|---|
| **Infrastructure encryption** | Second AES-256 layer with a different algorithm/key at the infrastructure level (Storage `requireInfrastructureEncryption`, SQL, managed disks with platform keys + CMK) |
| **In transit** | TLS plus MACsec, or TLS over VPN/ExpressRoute encryption |

Microsoft positions double encryption for customers who need protection against compromise of one encryption layer; for most workloads one layer with good key management is sufficient. Infrastructure encryption must usually be set at resource creation.

### VM disk encryption

| Option | Status | Notes |
|---|---|---|
| **Server-side encryption (SSE)** | Default | Managed disks, snapshots and images with platform keys; CMK via disk encryption sets |
| **Encryption at host** | Recommended | Encrypts temp disks and caches end to end on the host; works with PMK or CMK |
| **Confidential disk encryption** | For confidential VMs | OS disk encryption bound to the VM's TPM |
| **Azure Disk Encryption (ADE)** | **Retiring 15 September 2028** | BitLocker/DM-Crypt inside the guest; migrate ADE VMs (and backups) to encryption at host before retirement |

### Confidential computing

Protects data **in use** by running workloads in hardware-based trusted execution environments with **attestation** (Microsoft Azure Attestation). Options include confidential VMs, confidential containers on AKS/ACI, confidential GPUs for AI, Azure SQL Always Encrypted with secure enclaves, and Managed HSM-based secure key release. Use when you must protect data from cloud operators, multi-party analytics or sensitive AI inference.

## How it works on Azure

Storage account with CMK, infrastructure encryption and TLS 1.2 minimum:

```bicep
resource uai 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' existing = { name: 'id-st-cmk' }

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'stcontosodataprod'
  location: resourceGroup().location
  sku: { name: 'Standard_ZRS' }
  kind: 'StorageV2'
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uai.id}': {} } }
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    publicNetworkAccess: 'Disabled'
    encryption: {
      requireInfrastructureEncryption: true
      keySource: 'Microsoft.Keyvault'
      identity: { userAssignedIdentity: uai.id }
      keyvaultproperties: {
        keyvaulturi: 'https://kv-contoso-cmk-prod.vault.azure.net'
        keyname: 'cmk-storage'          // no version: auto-rotation picks up new versions
      }
      services: { blob: { enabled: true }, file: { enabled: true } }
    }
  }
}
```

The identity needs **Key Vault Crypto Service Encryption User** on the key; the vault needs soft delete and purge protection.

Enable encryption at host for a VM (subscription feature registration may be required):

```bash
az vm update -g rg-app -n vm-app01 --set securityProfile.encryptionAtHost=true
```

Enforce with Azure Policy: minimum TLS version for storage, App Service and SQL; CMK required for specific services; "encryption at host should be enabled" for VMs.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Default vs CMK** | Platform keys unless regulation, contract or a revocation requirement demands CMK. CMK adds operational risk (key loss = data loss) and dependencies. |
| **Key location** | Key Vault Premium for HSM-protected CMK; Managed HSM for single-tenant and stricter control. |
| **Key per service vs shared** | Separate keys per workload/data classification to limit revocation impact. |
| **Double encryption** | Only when required by policy or threat model; decide at creation time. |
| **In-use protection** | Confidential computing for highly sensitive processing; validate performance and feature support. |
| **TLS** | Enforce TLS 1.2+ everywhere; plan TLS 1.3 where supported; retire legacy clients. |
| **ADE migration** | Inventory ADE VMs now; migrate to encryption at host well before September 2028. |

## Hands-on

1. List services in your estate and whether they use platform keys or CMK (Resource Graph).
2. Create a storage account with CMK and infrastructure encryption; rotate the key and confirm the new version is used.
3. Enable encryption at host on a test VM; identify any ADE-encrypted VMs and plan migration.
4. Assign Azure Policy enforcing minimum TLS 1.2 for storage and App Service.
5. Deploy a confidential VM and review its attestation.

## Security, cost and reliability notes

- **Key availability is data availability**: if the key is deleted or access revoked, the service can't decrypt; protect keys with purge protection and RBAC, and monitor key access.
- **Crypto-shredding**: revoking CMK access is a valid way to render data unreadable at decommissioning; document the procedure.
- **Cost**: CMK adds Key Vault operations; Managed HSM and confidential computing sizes cost more.
- **Feature trade-offs**: some services limit features with CMK or client-side encryption (search, indexing, some replication options).

## Common mistakes

| Mistake | Fix |
|---|---|
| Believing data is unencrypted by default | Azure encrypts at rest by default |
| CMK everywhere "for security" | Use where required; manage the risk |
| Versioned key URIs that block rotation | Versionless URIs |
| Leaving ADE in place | Migrate to encryption at host before 15 September 2028 |
| Allowing TLS 1.0/1.1 clients | Enforce TLS 1.2+ |

## Check yourself

1. How is data protected at rest, in transit and in use on Azure by default?
2. Compare Microsoft-managed keys, CMK and client-side encryption.
3. What is infrastructure (double) encryption, and when is it justified?
4. What replaces Azure Disk Encryption, and by when?
5. What does confidential computing protect, and how is trust established?
6. What is the main operational risk of CMK?

<details>
<summary>Answers</summary>

1. At rest: service-side AES-256 with Microsoft-managed keys; in transit: TLS 1.2+ to services and MACsec between datacentres; in use: not protected unless you use confidential computing.
2. Microsoft-managed: no effort; CMK: you control keys in Key Vault/Managed HSM and can revoke; client-side: your app encrypts so the service never sees plaintext.
3. A second encryption layer at the infrastructure level with different keys/algorithms; justified when policy or threat model requires protection against one layer's compromise.
4. Encryption at host (or confidential disk encryption for confidential VMs); ADE retires on 15 September 2028.
5. Data in use, inside hardware trusted execution environments, verified through attestation.
6. Losing or revoking the key makes data unrecoverable or services unavailable.

</details>

## References

- [Azure encryption overview](https://learn.microsoft.com/azure/security/fundamentals/encryption-overview)
- [Encryption at rest](https://learn.microsoft.com/azure/security/fundamentals/encryption-atrest)
- [Data encryption models](https://learn.microsoft.com/azure/security/fundamentals/encryption-models)
- [Double encryption](https://learn.microsoft.com/azure/security/fundamentals/double-encryption)
- [Services supporting customer-managed keys](https://learn.microsoft.com/azure/security/fundamentals/encryption-customer-managed-keys-support)
- [Azure confidential computing](https://learn.microsoft.com/azure/confidential-computing/overview)
- [Disk encryption overview](https://learn.microsoft.com/azure/virtual-machines/disk-encryption-overview)
