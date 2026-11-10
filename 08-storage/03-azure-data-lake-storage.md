# Azure Data Lake Storage

## Why it matters

Azure Data Lake Storage (ADLS, formerly ADLS Gen2) is Blob Storage with a **hierarchical namespace**: real directories, atomic renames and POSIX-style access control lists. It is the storage layer for Azure Databricks, Synapse, HDInsight and many Fabric scenarios (through OneLake shortcuts), and a common home for AI training data and RAG source corpora. A well-designed lake makes analytics and AI fast, governed and cheap; a badly designed one becomes a data swamp with leaky permissions.

## Key concepts

### What hierarchical namespace changes

| | Flat Blob namespace | Hierarchical namespace (HNS) |
|---|---|---|
| Directories | Virtual (a `/` in the name) | Real objects |
| Rename or move a directory | Copy and delete every blob | Single atomic metadata operation |
| Delete a directory | Delete every blob | Single operation |
| Access control | RBAC (and SAS) | RBAC **plus POSIX ACLs** on directories and files |
| Endpoint | `blob.core.windows.net` | Also `dfs.core.windows.net` (ABFS driver: `abfss://`) |
| Analytics engines | Work, slower for directory operations | Optimised (Spark, Databricks, Synapse) |

Most Blob features now work with HNS, but a few have restrictions; check the feature support table. HNS can be enabled on an existing account through a one-way upgrade, after validation.

### Access control: RBAC and ACLs together

Authorisation is evaluated in order:

1. **Azure RBAC data roles** (`Storage Blob Data Owner`, `Contributor`, `Reader`) at account or container scope. If RBAC grants the operation, access is allowed.
2. If not, **POSIX ACLs** on the path are checked.

ACL details:

| Concept | Detail |
|---|---|
| Permissions | Read (r), write (w), execute (x). Execute on a directory is needed to traverse it |
| Entries | Owning user, owning group, named users, named groups (Entra security groups), service principals and managed identities, mask, other |
| **Access ACL** | Controls access to that object |
| **Default ACL** | On directories only: template inherited by new children. Not applied to existing children |
| Traversal | Every parent directory needs `--x` for the principal to reach a child |
| Limit | A maximum of 32 ACL entries per file or directory (including the base ones), so **always use groups**, not individual users |

To apply ACL changes to existing children, use recursive ACL updates (`az storage fs access update-recursive`).

### Lake zones (medallion architecture)

```text
abfss://lake@stdatalake.dfs.core.windows.net/
 ├─ bronze/   (raw, as ingested, immutable; partitioned by source and date)
 │    └─ crm/accounts/ingest_date=2026-10-15/part-0001.json
 ├─ silver/   (cleaned, conformed, deduplicated; Delta or Parquet)
 │    └─ sales/orders/
 ├─ gold/     (business-level aggregates and products for BI and AI)
 │    └─ finance/revenue_by_month/
 └─ sandbox/  (exploration, short retention)
```

| Practice | Why |
|---|---|
| Separate containers (or accounts) per zone | Different access, lifecycle and protection settings |
| Open formats: **Delta Lake**, **Apache Iceberg**, Parquet | Columnar, compressed, transactional, readable by many engines |
| Partition by date or key used in filters | Prune reads; avoid thousands of tiny partitions |
| Target file sizes of hundreds of MB | Small files kill performance ("small files problem"); compact regularly |
| Raw data immutable | Reprocessing remains possible |
| Data products owned by domains | Clear ownership in larger organisations (data mesh) |

### Relationship with Microsoft Fabric and OneLake

Microsoft Fabric stores data in **OneLake**, itself built on ADLS. Rather than copying data, Fabric can create **shortcuts** to existing ADLS containers, so a lake in your storage account can be queried by Fabric engines in place. Many organisations keep landing and raw data in ADLS accounts they control and expose curated data to Fabric through shortcuts (topic 12).

## How it works on Azure

### Creating and using a lake

```bash
az storage account create -g $RG -n $LAKE -l $LOC --kind StorageV2 --sku Standard_ZRS \
  --enable-hierarchical-namespace true --allow-shared-key-access false -o none

az storage fs create -n bronze --account-name $LAKE --auth-mode login
az storage fs directory create -f bronze -n crm/accounts --account-name $LAKE --auth-mode login
```

Spark (Databricks, Synapse, Fabric) reads with `abfss://bronze@<account>.dfs.core.windows.net/crm/accounts/`, authenticating with Entra (managed identity, service principal or user passthrough), not account keys.

### Python with the Data Lake SDK

```python
from azure.identity import DefaultAzureCredential
from azure.storage.filedatalake import DataLakeServiceClient

svc = DataLakeServiceClient("https://<lake>.dfs.core.windows.net", credential=DefaultAzureCredential())
fs = svc.get_file_system_client("bronze")
d = fs.get_directory_client("crm/accounts/ingest_date=2026-10-15")
d.create_directory()
f = d.get_file_client("part-0001.json")
f.upload_data(b'{"id": 1}', overwrite=True)

# Atomic rename of a whole directory
d.rename_directory("bronze/crm/accounts/ingest_date=2026-10-15-validated")
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Enable HNS | Yes for analytics, data engineering and large AI datasets; not necessary for simple object storage or static content |
| RBAC vs ACL | RBAC at container level for broad roles (engineering, platform); ACLs for fine-grained folder access by Entra groups |
| One account or several | Separate accounts for distinct security domains, regions or very high scale; containers per zone within an account |
| Format | Delta or Iceberg for tables; Parquet for files; keep JSON and CSV in bronze only |
| Fabric | Use shortcuts to avoid copies; decide where the system of record lives |
| Governance | Register the lake in Microsoft Purview for catalogue, classification and lineage |

## Hands-on

```bash
RG=rg-lake-lab; LOC=swedencentral; LAKE=stlake$RANDOM
az group create -n $RG -l $LOC -o none
az storage account create -g $RG -n $LAKE -l $LOC --sku Standard_LRS \
  --enable-hierarchical-namespace true --allow-shared-key-access false -o none
ME=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee $ME --role "Storage Blob Data Owner" \
  --scope $(az storage account show -g $RG -n $LAKE --query id -o tsv) -o none
# (role assignments can take a minute to apply)

# 1. Zones and folders
for z in bronze silver gold; do az storage fs create -n $z --account-name $LAKE --auth-mode login -o none; done
az storage fs directory create -f silver -n finance --account-name $LAKE --auth-mode login -o none
az storage fs directory create -f silver -n hr --account-name $LAKE --auth-mode login -o none

# 2. Give an Entra group read access to silver/finance only, via ACLs
GROUP_ID=$(az ad group create --display-name "lake-finance-readers-$RANDOM" --mail-nickname "lfr$RANDOM" --query id -o tsv)
az storage fs access update-recursive -f silver -p / --acl "group:$GROUP_ID:--x" --account-name $LAKE --auth-mode login
az storage fs access set -f silver -p finance --acl "user::rwx,group::r-x,other::---,group:$GROUP_ID:r-x,default:group:$GROUP_ID:r-x" \
  --account-name $LAKE --auth-mode login
az storage fs access show -f silver -p finance --account-name $LAKE --auth-mode login

# 3. Atomic directory rename
az storage fs directory move -f silver -n hr --new-directory silver/people --account-name $LAKE --auth-mode login
az storage fs file list -f silver --account-name $LAKE --auth-mode login -o table

az ad group delete --group $GROUP_ID
az group delete -n $RG --yes --no-wait
```

Note that the group needs execute (`--x`) on the root to traverse to `finance`, and that the default ACL makes new files inherit the permission.

## Security, cost and reliability notes

- **Security:** use Entra groups in ACLs, never individual users; manage group membership through access packages (topic 02).
- **Security:** remember that RBAC grants override ACLs. A broad `Storage Blob Data Reader` at account level silently grants read to everything.
- **Security:** private endpoints for both `blob` and `dfs` sub-resources; analytics engines often use `dfs`.
- **Cost:** HNS has a small premium on some transactions and metadata; usually negligible against performance gains.
- **Cost:** small files multiply transaction costs and slow queries. Compact.
- **Reliability:** protect bronze with soft delete and, for regulated data, immutability. Replay from bronze is your recovery path for silver and gold.

## Common mistakes

- Granting account-level RBAC and expecting folder ACLs to restrict access.
- Using individual users in ACLs and hitting the 32-entry limit.
- Setting default ACLs and assuming existing files changed too.
- Forgetting execute permission on parent directories.
- Storing analytics data as millions of small JSON files.
- Only creating a private endpoint for `blob`, then Spark jobs fail on `dfs`.
- Copying data into Fabric when a shortcut would do.

## Check yourself

1. What does hierarchical namespace add to Blob Storage?
2. In what order are RBAC and ACLs evaluated?
3. What is the difference between an access ACL and a default ACL?
4. Why does a user with read on a file still get "forbidden" when reading it?
5. Why use groups rather than users in ACLs?
6. What are the bronze, silver and gold zones for?
7. How can Microsoft Fabric use data in an existing ADLS account without copying it?

<details>
<summary>Answers</summary>

1. Real directories with atomic rename and delete, POSIX ACLs, and the `dfs` endpoint used by analytics engines.
2. RBAC first; if RBAC allows the operation, access is granted. Otherwise ACLs are checked.
3. An access ACL governs access to the object itself; a default ACL on a directory is the template copied to new children.
4. They probably lack execute permission on one of the parent directories, which is needed to traverse the path.
5. There is a limit of 32 ACL entries per object, and group membership is easier to manage and audit than editing ACLs.
6. Bronze holds raw, immutable ingested data; silver holds cleaned and conformed data; gold holds business-ready aggregates and data products.
7. With OneLake shortcuts pointing at the ADLS location.

</details>

## References

- [Azure Data Lake Storage introduction](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction)
- [Hierarchical namespace](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-namespace)
- [Access control model](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-access-control-model)
- [ACLs in Azure Data Lake Storage](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-access-control)
- [Best practices for using Data Lake Storage](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-best-practices)
- [Blob Storage feature support with HNS](https://learn.microsoft.com/azure/storage/blobs/storage-feature-support-in-storage-accounts)
- [OneLake shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-shortcuts)
- [Medallion lakehouse architecture](https://learn.microsoft.com/azure/databricks/lakehouse/medallion)
