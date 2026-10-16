# Storage Actions, Discovery and Insights

> Topic: 08 · Storage · Tracks: E – A ○ C ○ · Status checked: 2026-10

## Why it matters

Large organisations have hundreds of storage accounts holding billions of objects. Questions such as "how much data do we have, where, and how much of it is cold?", "which accounts still allow shared key access?" and "can we retag or re-tier ten million blobs?" cannot be answered by scripts that list every blob. Azure provides estate-wide visibility (Storage Discovery and Storage Insights) and serverless data management at scale (Storage Actions). These are the tools behind storage cost optimisation and governance work, often led by consultants and architects.

## Key concepts

### Azure Storage Actions

A serverless framework for performing operations on millions or billions of objects across storage accounts, without writing or hosting code.

```text
Storage task  (conditions + operations, defined once)
   │  assigned to
   ▼
Storage task assignments  → account A (prefix "raw/"), account B, account C
   │  run once or on a schedule
   ▼
Execution reports (which objects matched, what was done, failures)
```

| Element | Detail |
|---|---|
| **Conditions** | Evaluate blob properties: name, prefix, size, access tier, creation and modified time, last access time, index tags, metadata, version state, immutability |
| **Operations** | Set tier, set or delete index tags, set immutability policy or legal hold, delete blob, undelete, copy (check current list) |
| **Assignments** | Apply a task to a storage account with a filter, run once or recurring |
| **Identity** | The task uses a managed identity with data roles on target accounts |
| **Preview** | A preview runs the conditions against sample data before execution |
| **Reports** | CSV reports of evaluated and processed objects to a container |

Example uses:

- Tag every blob with `classification=confidential` when its name matches a pattern, so ABAC and Defender can use the tag.
- Apply legal holds to all objects for a litigation case across several accounts.
- Move blobs not read in 180 days to cold, beyond what simple lifecycle rules can express.
- Clean up untagged temporary data from AI experiments.

Storage Actions complements **lifecycle management** (file 02): lifecycle rules are simple, free and continuous within one account; Storage Actions handles richer conditions, more operations and multiple accounts, at a per-operation price.

### Azure Storage Discovery

An estate-wide service that aggregates metadata and metrics from storage accounts across subscriptions and tenants into a single workspace.

| Capability | Detail |
|---|---|
| Scope | Management groups, subscriptions or resource groups |
| Insights | Capacity and object counts over time, distribution by tier, redundancy, region, account type |
| Security posture | Accounts allowing public access, shared key, missing encryption settings, network exposure |
| Activity | Transaction trends, top accounts by growth |
| Natural language | Copilot-based questions over the estate ("which accounts grew most last month?"); check current status |
| Retention | Historical trends for planning |

Use it to answer estate-level questions and find optimisation and risk hotspots, then act with Storage Actions, policy or lifecycle rules.

### Storage Insights and monitoring

| Tool | Scope | What it shows |
|---|---|---|
| **Storage Insights** (Azure Monitor workbook) | One or many accounts | Capacity, transactions, latency, availability, errors from platform metrics |
| **Metrics** | Account and service | `UsedCapacity`, `Transactions` (by API and response type), `SuccessE2ELatency`, `SuccessServerLatency`, `Availability`, `Egress`, `Ingress` |
| **Resource logs** | Per request | `StorageRead`, `StorageWrite`, `StorageDelete` with caller identity, auth type, IP |
| **Blob inventory** | Per account | Daily or weekly listing of every blob and its properties to CSV or Parquet, queryable with Synapse, Databricks or Fabric |
| **Azure Advisor** | Subscription | Cost and reliability recommendations |
| **Cost Management** | Billing scope | Spend by meter: capacity, transactions, data retrieval, geo-replication |

Latency tip: a large gap between end-to-end latency and server latency points to the client or network, not the storage service.

## How it works on Azure

### A storage task in practice (portal flow)

1. Create a **Storage task**: name, managed identity, region.
2. Define **conditions**, for example: `Blob.Tags['ingest'] == 'done'` AND `Blob.LastAccessedOn < 120 days ago`.
3. Define **operations**: set tier to cold; set tag `archived=true`.
4. **Preview** against sample blobs.
5. Grant the task's managed identity `Storage Blob Data Owner` on target accounts.
6. Create **assignments** to accounts with prefix filters and a schedule.
7. Review execution reports.

### Querying blob inventory

Enable inventory with Parquet output, then query with any engine:

```sql
-- Fabric or Synapse serverless SQL over the inventory output
SELECT AccessTier, COUNT(*) AS blobs, SUM([Content-Length]) / POWER(1024.0, 4) AS TiB
FROM OPENROWSET(BULK 'https://<acct>.blob.core.windows.net/inventory/2026/10/15/*/*.parquet', FORMAT = 'PARQUET') AS r
GROUP BY AccessTier;
```

### Detecting shared key use before disabling it

```kusto
StorageBlobLogs
| where TimeGenerated > ago(14d)
| summarize requests = count() by AccountName, AuthenticationType, CallerIpAddress, UserAgentHeader
| where AuthenticationType in ("AccountKey", "SAS")
| order by requests desc
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Lifecycle rules vs Storage Actions | Lifecycle for age-based tiering and deletion within an account; Storage Actions for complex conditions, tags, holds and multi-account operations |
| Discovery vs custom scripts | Discovery for estate-wide reporting; avoid writing list-every-blob scripts against large accounts |
| Inventory | Enable on large accounts where you need object-level analysis; output Parquet |
| Logging | Resource logs to Log Analytics for security-sensitive accounts; sample or archive to storage for high-volume accounts to control cost |

## Hands-on

1. Enable **blob inventory** on a lab account (daily, Parquet, all blobs with tier, size, last access) and inspect the output the next day.
2. Open **Storage Insights** in Azure Monitor across all your subscriptions and identify the account with the highest transaction count.
3. Create a **Storage task** that tags any blob larger than 10 MB with `size=large`, preview it, assign it to a lab account and review the report.
4. Run the KQL above against an account with diagnostic logs enabled to see which authentication types are in use.

```bash
# Inventory policy via CLI
cat > inventory.json <<'EOF'
{ "enabled": true, "type": "Inventory",
  "rules": [ { "enabled": true, "name": "daily", "destination": "inventory",
    "definition": { "format": "Parquet", "schedule": "Daily", "objectType": "Blob",
      "schemaFields": [ "Name", "Content-Length", "AccessTier", "LastAccessTime", "Tags" ],
      "filters": { "blobTypes": [ "blockBlob" ] } } } ] }
EOF
az storage container create --account-name $SA -n inventory --auth-mode login -o none
az storage account blob-inventory-policy create -g $RG --account-name $SA --policy @inventory.json
```

## Security, cost and reliability notes

- **Security:** storage tasks act with broad data permissions on many accounts; scope assignments tightly and review conditions with preview before running destructive operations such as delete.
- **Security:** use Discovery's posture views as a recurring report for shared key, public access and network exposure.
- **Cost:** Storage Actions is billed per task execution and objects processed; lifecycle management is free (you pay only for resulting operations).
- **Cost:** resource logs for busy accounts can cost more than the storage. Use filters, basic logs tables or archiving.
- **Reliability:** run destructive tasks in report-only or preview mode first; enable soft delete as a safety net.

## Common mistakes

- Writing scripts that enumerate billions of blobs instead of using inventory.
- Running a delete task across accounts without preview or soft delete.
- Disabling shared key without checking logs for current key usage.
- Lifecycle rules on last access time without enabling access time tracking.
- No estate-wide view, so cost optimisation happens account by account.

## Check yourself

1. What are the parts of a storage task?
2. When would you use Storage Actions instead of lifecycle management?
3. What questions does Storage Discovery answer?
4. How can you find which clients still use account keys?
5. What does a large gap between end-to-end and server latency indicate?
6. Why use blob inventory instead of listing blobs?

<details>
<summary>Answers</summary>

1. Conditions, operations, a managed identity, and assignments to storage accounts with filters and schedules, plus execution reports.
2. When you need richer conditions (tags, metadata, multiple properties), operations lifecycle rules do not support (tagging, legal holds), or the same operation across many accounts.
3. Estate-wide questions about capacity, growth, tier and redundancy distribution, activity, and security posture across subscriptions.
4. Query resource logs (`StorageBlobLogs`) for `AuthenticationType` of `AccountKey` or `SAS`, grouped by caller.
5. Time is being spent in the client or network rather than in the storage service.
6. Inventory is generated by the service on a schedule and output as files you can query efficiently, avoiding slow and costly list operations at scale.

</details>

## References

- [Azure Storage Actions overview](https://learn.microsoft.com/azure/storage-actions/overview)
- [Storage task conditions and operations](https://learn.microsoft.com/azure/storage-actions/storage-tasks/storage-task-conditions-operations)
- [Azure Storage Discovery](https://learn.microsoft.com/azure/storage-discovery/overview)
- [Storage Insights](https://learn.microsoft.com/azure/storage/common/storage-insights-overview)
- [Monitor Blob Storage](https://learn.microsoft.com/azure/storage/blobs/monitor-blob-storage)
- [Blob inventory](https://learn.microsoft.com/azure/storage/blobs/blob-inventory)
