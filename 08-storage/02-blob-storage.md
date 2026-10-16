# Blob Storage

> Topic: 08 · Storage · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Blob Storage is Azure's object store and probably its most widely used service. It holds web content, user uploads, backups, logs, data lake files, training sets, model artefacts, and the source documents behind most RAG systems. Engineers need to read and write it efficiently and securely; architects need to design tiering, retention and protection so that data is cheap to keep and hard to lose.

## Key concepts

### Structure

```text
Storage account
 └─ Container  (like a bucket; access control boundary for some features)
     └─ Blob   "reports/2026/10/q3.pdf"   ← "/" is part of the name unless hierarchical namespace is on
          ├─ properties, metadata (key-value), index tags
          ├─ versions, snapshots
          └─ access tier
```

### Blob types

| Type | Structure | Use |
|---|---|---|
| **Block blob** | Up to 50,000 blocks, committed as a list; very large maximum size | Almost everything: files, media, datasets, models |
| **Append blob** | Blocks only appended | Logs |
| **Page blob** | 512-byte pages, random read and write | Disks (managed disks use page blobs internally) |

Large uploads are split into blocks uploaded in parallel and committed at the end; SDKs do this automatically.

### Access tiers and lifecycle management

Tiers (hot, cool, cold, archive) are covered in file 01. **Lifecycle management** policies apply rules to blobs by prefix, index tag or blob type:

```json
{
  "rules": [
    {
      "name": "age-out-raw",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": [ "blockBlob" ], "prefixMatch": [ "raw/" ] },
        "actions": {
          "baseBlob": {
            "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
            "tierToCold":    { "daysAfterModificationGreaterThan": 90 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
            "delete":        { "daysAfterModificationGreaterThan": 2555 }
          },
          "version":  { "delete": { "daysAfterCreationGreaterThan": 90 } },
          "snapshot": { "delete": { "daysAfterCreationGreaterThan": 90 } }
        }
      }
    }
  ]
}
```

Conditions can also use `daysAfterLastAccessTimeGreaterThan` if **last access time tracking** is enabled, which is better for data whose value depends on use rather than age.

### Data protection features

| Feature | Protects against | Notes |
|---|---|---|
| **Soft delete for blobs** | Accidental delete or overwrite | Deleted blobs recoverable for 1–365 days |
| **Soft delete for containers** | Container deletion | Separate setting |
| **Versioning** | Overwrites and deletes | Every write keeps the previous version; works with lifecycle rules to expire old versions |
| **Point-in-time restore** | Bulk corruption or deletion | Restore block blobs in a range to an earlier time; requires versioning, change feed and soft delete |
| **Snapshots** | Manual point-in-time copies | Mostly superseded by versioning |
| **Change feed** | Audit and processing | Ordered, durable log of changes |
| **Immutable storage (WORM)** | Tampering and deletion, regulatory retention | Time-based retention and legal holds, at container or version level; locked policies cannot be shortened, even by Microsoft |
| **Resource locks** | Account deletion | `CanNotDelete` lock on the account |
| **Azure Backup for blobs** | Operational and vaulted backup | Vaulted backup copies data to a separate vault for ransomware resilience |

A sensible default for important data: soft delete (blobs and containers), versioning with lifecycle cleanup, change feed, and a delete lock on the account. Add immutability or vaulted backup for regulated or ransomware-sensitive data.

### Object replication

Asynchronously copies block blobs from a source container to a destination container in another account (any region), by rules and prefixes. Uses: keep a copy near users in another region, feed a separate analytics account, or meet data segregation needs. It requires versioning and change feed. Unlike GRS, the destination is readable and writable as a normal account, and you choose what to replicate.

### Finding data

| Feature | Use |
|---|---|
| **Blob index tags** | Key-value tags on blobs, queryable across a container or account (`az storage blob filter`) |
| **Metadata** | Key-value pairs returned with the blob; not queryable server-side |
| **Blob inventory** | Daily or weekly reports of all blobs to CSV or Parquet |
| **Change feed** | Event history for processing pipelines |
| **Event Grid** | Events on blob create and delete for event-driven processing |

### Other capabilities

- **Static website hosting** from a `$web` container (often fronted by Front Door).
- **SFTP** and **NFS 3.0** protocol support (with hierarchical namespace), for legacy integration.
- **Encryption scopes** to use different keys per container or blob.

## How it works on Azure

### Uploading and downloading from Python (keyless)

```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient, StandardBlobTier

svc = BlobServiceClient("https://<account>.blob.core.windows.net", credential=DefaultAzureCredential())
container = svc.get_container_client("docs")

with open("handbook.pdf", "rb") as f:
    container.upload_blob(
        name="hr/handbook.pdf", data=f, overwrite=True,
        metadata={"department": "hr", "classification": "internal"},
        tags={"dept": "hr", "ingest": "pending"},
        standard_blob_tier=StandardBlobTier.HOT,
        max_concurrency=4,
    )

# Query by index tag across the account
for b in svc.find_blobs_by_tags("\"dept\"='hr' AND \"ingest\"='pending'"):
    print(b.container_name, b.name)

data = container.download_blob("hr/handbook.pdf", max_concurrency=4).readall()
```

The identity needs `Storage Blob Data Contributor` (write) or `Reader` (read) on the container or account.

### Event-driven processing

```text
Blob created ─► Event Grid ─► Service Bus queue / Function / Container Apps job
                                   └─► extract, chunk, embed, index (RAG ingestion)
```

Use Event Grid rather than polling. For high volumes, route events to a queue so processors can scale and retry.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Container design | By access boundary and lifecycle (for example `raw`, `curated`, `public-assets`), not one container for everything |
| Naming | Prefixes that support lifecycle rules and parallelism (date or tenant prefixes) |
| Tiering | Hot for active data; lifecycle to cool and cold; archive only for data you will rarely and slowly need |
| Protection | Soft delete and versioning on by default; immutability for regulated records; vaulted backup for critical data |
| Versioning cost | Every overwrite keeps a version; add lifecycle rules to expire old versions |
| Object replication vs GRS | GRS for whole-account DR; object replication for selective, readable copies |
| Metadata vs index tags | Index tags when you need to query; metadata for descriptive data read with the blob |

## Hands-on

```bash
RG=rg-blob-lab; LOC=swedencentral; SA=stblob$RANDOM
az group create -n $RG -l $LOC -o none
az storage account create -g $RG -n $SA -l $LOC --sku Standard_ZRS --allow-shared-key-access false -o none
ME=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee $ME --role "Storage Blob Data Contributor" \
  --scope $(az storage account show -g $RG -n $SA --query id -o tsv) -o none

# 1. Protection settings
az storage account blob-service-properties update -g $RG -n $SA \
  --enable-versioning true --enable-change-feed true \
  --enable-delete-retention true --delete-retention-days 14 \
  --enable-container-delete-retention true --container-delete-retention-days 14 \
  --enable-last-access-tracking true -o none

# 2. Upload, overwrite, and list versions
az storage container create --account-name $SA -n docs --auth-mode login -o none
echo "v1" > note.txt; az storage blob upload --account-name $SA -c docs -n note.txt -f note.txt --auth-mode login -o none
echo "v2" > note.txt; az storage blob upload --account-name $SA -c docs -n note.txt -f note.txt --overwrite --auth-mode login -o none
az storage blob list --account-name $SA -c docs --include v --auth-mode login --query "[].{name:name, version:versionId, current:isCurrentVersion}" -o table

# 3. Delete and recover
az storage blob delete --account-name $SA -c docs -n note.txt --auth-mode login
az storage blob list --account-name $SA -c docs --include dv --auth-mode login -o table

# 4. Lifecycle policy (save the JSON above as policy.json first)
az storage account management-policy create -g $RG --account-name $SA --policy @policy.json -o none

# 5. Index tags
az storage blob upload --account-name $SA -c docs -n hr/a.txt -f note.txt --tags dept=hr --auth-mode login -o none
az storage blob filter --account-name $SA --tag-filter "\"dept\"='hr'" --auth-mode login -o table

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** disable anonymous public access at the account level; serve public content through Front Door or a CDN.
- **Security:** use Entra roles scoped to containers; avoid account keys and long-lived SAS (file 06).
- **Cost:** versioning plus frequent overwrites can multiply storage. Expire old versions with lifecycle rules.
- **Cost:** archive has early deletion charges and expensive, slow rehydration. Do not archive data you might need quickly.
- **Cost:** many small objects cost more in transactions than in capacity. Batch small items where you can.
- **Reliability:** soft delete and versioning are the first line of defence against human error and many ransomware scenarios; immutability or vaulted backup is the last.

## Common mistakes

- Turning on versioning without lifecycle rules for old versions, then a large bill.
- Archiving data needed for an urgent restore.
- Polling containers for new files instead of using Event Grid.
- Relying on metadata for search when index tags were needed.
- Public containers for "temporary" sharing.
- No soft delete, so an accidental script deletes data permanently.
- Lifecycle rules based on modification time for data that is read often but rarely written (use last access time).

## Check yourself

1. What are the three blob types and their uses?
2. Which features must be enabled for point-in-time restore?
3. How does versioning differ from soft delete?
4. What does a locked immutability policy guarantee?
5. When would you use object replication instead of GRS?
6. What is the difference between blob index tags and metadata?
7. How should a RAG ingestion pipeline find newly uploaded documents?

<details>
<summary>Answers</summary>

1. Block blobs for general files and data; append blobs for logs; page blobs for random-access disks.
2. Versioning, change feed and blob soft delete.
3. Versioning keeps every previous version on each write; soft delete keeps deleted blobs (and overwritten data, if versioning is off) for a retention period.
4. Data cannot be modified or deleted until the retention period ends, and the policy cannot be shortened or removed, even by administrators or Microsoft.
5. When you need a selectively chosen, readable and writable copy in another account or region, rather than whole-account DR replication.
6. Index tags can be queried across containers and an account; metadata is returned with the blob but is not queryable server-side.
7. Subscribe to Blob Created events through Event Grid, typically into a queue that triggers processors.

</details>

## References

- [Blob Storage introduction](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-introduction)
- [Lifecycle management](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview)
- [Data protection overview](https://learn.microsoft.com/azure/storage/blobs/data-protection-overview)
- [Blob versioning](https://learn.microsoft.com/azure/storage/blobs/versioning-overview)
- [Immutable storage](https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview)
- [Object replication](https://learn.microsoft.com/azure/storage/blobs/object-replication-overview)
- [Blob index tags](https://learn.microsoft.com/azure/storage/blobs/storage-manage-find-blobs)
- [Azure Storage Blob client library for Python](https://learn.microsoft.com/python/api/overview/azure/storage-blob-readme)
