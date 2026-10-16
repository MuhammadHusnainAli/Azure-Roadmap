# Storage Security

> Topic: 08 · Storage · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Exposed storage accounts are one of the most common causes of cloud data breaches: public containers, leaked account keys, long-lived SAS tokens in source code. AI raises the stakes, because storage now holds the documents that RAG systems index and the data that models are trained on. A secure storage account uses identity instead of keys, private networking instead of the internet, encryption with appropriate key control, and threat detection. All of it is configuration you can enforce with policy.

## Key concepts

### Authorisation options

| Method | How it works | Recommendation |
|---|---|---|
| **Microsoft Entra ID + RBAC** | OAuth tokens for users, groups, managed identities; data-plane roles | **Default** for everything |
| **Shared Key** (account keys) | Two 512-bit keys grant full access to everything in the account | Disable (`allowSharedKeyAccess: false`) |
| **Account SAS** | Signed with an account key; access to one or more services | Avoid |
| **Service SAS** | Signed with an account key; one resource (container, blob, share, queue, table) | Avoid; use stored access policies if unavoidable |
| **User delegation SAS** | Signed with an Entra-issued user delegation key; Blob and Data Lake only | **The preferred SAS** when you must hand out a URL |
| **Anonymous public read** | No auth for containers set to public | Disable at account level |
| ACLs (Data Lake) | POSIX on directories and files | With RBAC (file 03) |

**Data-plane roles** are separate from management roles. `Owner` or `Contributor` on an account lets you manage it (and list keys, if shared key is enabled), but reading data with Entra requires a data role:

| Service | Roles |
|---|---|
| Blob / Data Lake | `Storage Blob Data Owner`, `Contributor`, `Reader`, `Storage Blob Delegator` (for user delegation SAS) |
| Files (SMB) | `Storage File Data SMB Share Reader`, `Contributor`, `Elevated Contributor` |
| Files (REST, privileged) | `Storage File Data Privileged Reader`, `Contributor` |
| Queue | `Storage Queue Data Contributor`, `Reader`, `Message Sender`, `Message Processor` |
| Table | `Storage Table Data Contributor`, `Reader` |

Assign at the narrowest scope possible: container or share, not account or subscription. Use **ABAC conditions** on role assignments to narrow further (for example, only blobs with a particular index tag or path prefix).

### Shared access signatures, safely

A SAS is a URL that carries its own authority. Anyone with it has access until it expires.

| Rule | Why |
|---|---|
| Prefer **user delegation SAS** | Revocable by revoking the delegation key or the user's access; tied to an Entra identity; auditable |
| **Short expiry** (minutes or hours) | Limits damage if leaked |
| Minimal permissions and scope | Read one blob, not write the account |
| HTTPS only, IP restrictions where possible | Reduces misuse |
| Stored access policies for service SAS | Lets you revoke by changing the policy |
| SAS expiration policy on the account | Flags or blocks SAS with long lifetimes |
| Never put SAS in source code or logs | They are credentials |

### Network controls

| Control | Effect |
|---|---|
| **Public network access: Disabled** | Only private endpoints work |
| **Firewall (selected networks)** | Allow listed VNets (service endpoints) and public IP ranges |
| **Private endpoints** | Private IP per sub-resource (`blob`, `dfs`, `file`, `queue`, `table`, `web`), with private DNS zones |
| **Resource instance rules** | Allow specific Azure resource instances (for example a particular AI Search service or Foundry resource) through the firewall using their managed identity |
| **Trusted Microsoft services exception** | Allows certain services to bypass the firewall; prefer resource instance rules for precision |
| **Network security perimeter** | Group PaaS resources in a perimeter with shared access rules (topic 05 file 16) |

### Encryption

| Layer | Detail |
|---|---|
| In transit | HTTPS required; **minimum TLS 1.2** (older TLS versions are no longer supported); SMB 3.x encryption |
| At rest (default) | AES-256 with **Microsoft-managed keys**, always on, no cost |
| **Customer-managed keys (CMK)** | Key in Key Vault or Managed HSM; the account uses a managed identity to wrap and unwrap. You control rotation and revocation |
| **Infrastructure encryption** | Second layer of encryption at the infrastructure level, chosen at creation |
| **Encryption scopes** | Different keys per container or blob, for multi-tenant separation |
| Customer-provided keys | Key sent on each request (Blob); rare |

CMK is a compliance control, not a magic security upgrade: it lets you crypto-shred data by revoking a key and proves key custody. It adds operational risk (lose the key, lose the data) and cost.

### Threat protection and monitoring

**Microsoft Defender for Storage**:

| Capability | Detail |
|---|---|
| Activity monitoring | Detects anomalous access, suspicious IPs, unusual data extraction, SAS abuse |
| **Malware scanning** | Scans blobs on upload (near real time) or on demand; results as index tags and alerts; can trigger quarantine through Event Grid |
| **Sensitive data threat detection** | Prioritises alerts on accounts holding sensitive data (with Defender CSPM) |

Also: diagnostic settings sending `StorageRead`, `StorageWrite`, `StorageDelete` logs to Log Analytics for audit, and Azure Policy for configuration compliance.

## How it works on Azure

### A locked-down account

```bash
az storage account create -g $RG -n $SA -l $LOC --kind StorageV2 --sku Standard_ZRS \
  --https-only true --min-tls-version TLS1_2 \
  --allow-blob-public-access false --allow-shared-key-access false \
  --default-action Deny --bypass None --public-network-access Disabled \
  --require-infrastructure-encryption true -o none

# Private endpoint for blob, with private DNS
az network private-endpoint create -g $RG -n pe-$SA-blob --vnet-name vnet-app --subnet snet-pe \
  --private-connection-resource-id $(az storage account show -g $RG -n $SA --query id -o tsv) \
  --group-id blob --connection-name blob -o none
# (then link privatelink.blob.core.windows.net and create the zone group)
```

### Issuing a user delegation SAS in Python

```python
from datetime import datetime, timedelta, timezone
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient, BlobSasPermissions, generate_blob_sas

account = "<account>"
svc = BlobServiceClient(f"https://{account}.blob.core.windows.net", credential=DefaultAzureCredential())
now = datetime.now(timezone.utc)
key = svc.get_user_delegation_key(now - timedelta(minutes=5), now + timedelta(hours=1))
sas = generate_blob_sas(
    account_name=account, container_name="exports", blob_name="report.pdf",
    user_delegation_key=key, permission=BlobSasPermissions(read=True),
    start=now - timedelta(minutes=5), expiry=now + timedelta(minutes=15), protocol="https",
)
print(f"https://{account}.blob.core.windows.net/exports/report.pdf?{sas}")
```

The calling identity needs `Storage Blob Delegator` (or a data role that includes it) plus read on the blob.

### Policies to enforce

| Built-in policy (names approximate) | Effect |
|---|---|
| Storage accounts should prevent shared key access | Audit or deny |
| Storage account public access should be disallowed | Deny |
| Storage accounts should have the specified minimum TLS version | Deny |
| Storage accounts should use private link | Audit |
| Storage accounts should disable public network access | Deny |
| Configure Defender for Storage to be enabled | DeployIfNotExists |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Keys or Entra | Entra everywhere. Check that every client (old tools, third-party apps, some services) supports it before disabling shared key |
| SAS | Only user delegation SAS, short-lived, for browser downloads and uploads or partner hand-offs |
| Network | Private endpoints and public access disabled for internal data; for internet-facing content, put Front Door in front instead of opening the account |
| CMK | When regulation or policy requires customer key custody; otherwise Microsoft-managed keys |
| Defender malware scanning | On for any account accepting uploads from users or partners, especially before RAG ingestion |
| Scope of role assignments | Container level; ABAC conditions for path or tag based rules |

## Hands-on

```bash
RG=rg-stsec-lab; LOC=swedencentral; SA=stsec$RANDOM
az group create -n $RG -l $LOC -o none
az storage account create -g $RG -n $SA -l $LOC --sku Standard_LRS --allow-blob-public-access false -o none
SA_ID=$(az storage account show -g $RG -n $SA --query id -o tsv)
ME=$(az ad signed-in-user show --query id -o tsv)

# 1. Key-based access works (bad)
az storage container create --account-name $SA -n test --auth-mode key -o none && echo "key auth works"

# 2. Disable shared key and retry
az storage account update -g $RG -n $SA --allow-shared-key-access false -o none
az storage container create --account-name $SA -n test2 --auth-mode key 2>&1 | tail -1      # fails

# 3. Entra auth fails until you have a data role (Owner is not enough)
az storage container create --account-name $SA -n test2 --auth-mode login 2>&1 | tail -1    # AuthorizationPermissionMismatch
az role assignment create --assignee $ME --role "Storage Blob Data Contributor" --scope $SA_ID -o none
sleep 60
az storage container create --account-name $SA -n test2 --auth-mode login -o none && echo "entra auth works"

# 4. A user delegation SAS valid for 15 minutes
echo "secret report" > r.txt
az storage blob upload --account-name $SA -c test2 -n r.txt -f r.txt --auth-mode login -o none
EXP=$(date -u -d "+15 minutes" '+%Y-%m-%dT%H:%MZ')
SAS=$(az storage blob generate-sas --account-name $SA -c test2 -n r.txt --permissions r \
  --expiry $EXP --auth-mode login --as-user -o tsv)
curl -s "https://$SA.blob.core.windows.net/test2/r.txt?$SAS"

# 5. Enable Defender for Storage on the account (billed)
# az security pricing create -n StorageAccounts --tier standard --subplan DefenderForStorageV2

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** the account key is a root credential. If shared key is enabled, anyone with `Microsoft.Storage/storageAccounts/listKeys/action` (including `Contributor`) can read all data.
- **Security:** rotate keys if they were ever used, then disable shared key.
- **Security:** scan uploads before indexing them into RAG; poisoned or malicious documents become part of an AI system's context.
- **Cost:** Defender for Storage is billed per account plus per GB scanned for malware scanning; cap scanning per month if needed.
- **Cost:** private endpoints have hourly and per-GB charges; usually small next to the risk reduction.
- **Reliability:** CMK with a deleted or disabled key makes data inaccessible. Enable soft delete and purge protection on the Key Vault.

## Common mistakes

- Assuming `Owner` or `Contributor` can read blobs with Entra auth (they need data roles).
- Disabling shared key before checking which tools or services still depend on it.
- Long-lived account SAS tokens committed to repositories.
- Private endpoint for `blob` only, with Data Lake or Files clients failing.
- Using "allow trusted Microsoft services" broadly instead of resource instance rules.
- CMK without Key Vault purge protection.
- No diagnostic logs, so nobody can tell who read what.

## Check yourself

1. Why should shared key access be disabled?
2. What is the difference between a service SAS and a user delegation SAS?
3. Which role does a user need to read blobs with Entra auth, and why is `Owner` not enough?
4. What are resource instance rules and why prefer them over the trusted services exception?
5. What does a customer-managed key give you, and what risk does it add?
6. What does Defender for Storage malware scanning do with results?
7. Name four Azure Policy controls you would enforce on storage accounts.

<details>
<summary>Answers</summary>

1. Account keys grant full access to all data, cannot be scoped or tied to an identity, and are often leaked. Entra auth is scoped, auditable and keyless.
2. A service SAS is signed with an account key; a user delegation SAS is signed with an Entra-issued key, tied to an identity, revocable and limited to Blob and Data Lake.
3. A data role such as `Storage Blob Data Reader`. `Owner` is a management-plane role; data access with Entra requires data-plane permissions.
4. Rules that allow specific Azure resource instances through the storage firewall based on their managed identity. They are precise, whereas the trusted services exception allows broad categories of services.
5. Control over the key's lifecycle, rotation and revocation (crypto-shredding) and proof of key custody. The risk is data loss or outage if the key is deleted, disabled or inaccessible.
6. Writes the verdict as a blob index tag, raises alerts for malicious files, and can send events to Event Grid for automated quarantine.
7. Any four of: prevent shared key access, disallow public blob access, minimum TLS version, disable public network access, require private link, enable Defender for Storage, require CMK where mandated, require secure transfer.

</details>

## References

- [Security recommendations for Blob Storage](https://learn.microsoft.com/azure/storage/blobs/security-recommendations)
- [Authorize access with Microsoft Entra ID](https://learn.microsoft.com/azure/storage/blobs/authorize-access-azure-active-directory)
- [Prevent Shared Key authorization](https://learn.microsoft.com/azure/storage/common/shared-key-authorization-prevent)
- [Grant limited access with SAS](https://learn.microsoft.com/azure/storage/common/storage-sas-overview)
- [Create a user delegation SAS](https://learn.microsoft.com/azure/storage/blobs/storage-blob-user-delegation-sas-create-python)
- [Configure Azure Storage firewalls and virtual networks](https://learn.microsoft.com/azure/storage/common/storage-network-security)
- [Customer-managed keys](https://learn.microsoft.com/azure/storage/common/customer-managed-keys-overview)
- [Microsoft Defender for Storage](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-storage-introduction)
- [ABAC conditions for Blob Storage](https://learn.microsoft.com/azure/storage/blobs/storage-auth-abac)
