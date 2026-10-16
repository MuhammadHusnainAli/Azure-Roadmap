# 08 · Storage

> Tracks: E ● A ● C ○ · Part B · Core Infrastructure and Application Platform

Almost every Azure solution stores something: documents, images, logs, datasets, backups, model checkpoints, the files behind a RAG index. Azure Storage is the foundation for much of it, and it also sits underneath other services such as Foundry, Azure Machine Learning, Data Lake analytics and Microsoft Fabric shortcuts. This topic covers storage accounts and their services (Blob, Data Lake, Files, Queues, Tables), how to secure them, the specialised file and block services for demanding workloads, how to move data in and out, and how to lay out storage for AI.

## What you will be able to do

- Choose a storage account type, performance tier and redundancy option, and explain the failover model.
- Design Blob Storage with access tiers, lifecycle rules, versioning, soft delete and immutability.
- Lay out a Data Lake for analytics and AI with hierarchical namespace and ACLs.
- Choose between Azure Files, Azure NetApp Files, Elastic SAN and Managed Lustre.
- Decide when Queue or Table Storage is enough and when Service Bus or Cosmos DB is needed.
- Lock down storage with Entra authorisation, private endpoints and Defender for Storage, with no shared keys.
- Pick the right tool to move data, from AzCopy to Data Box.
- Organise storage for RAG corpora, training data and Foundry projects.

## Prerequisites

[Topic 01](../01-cloud-and-azure-fundamentals/README.md) (regions, zones, SLAs), [Topic 02](../02-identity-and-access/README.md) (RBAC, managed identities) and [Topic 05](../05-networking/README.md) (private endpoints, DNS).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Storage Accounts](01-storage-accounts.md) | 50 min | CLI |
| 02 | [Blob Storage](02-blob-storage.md) | 60 min | CLI, Python |
| 03 | [Azure Data Lake Storage](03-azure-data-lake-storage.md) | 45 min | CLI |
| 04 | [Azure Files and File Sync](04-azure-files-and-file-sync.md) | 45 min | CLI |
| 05 | [Queue and Table Storage](05-queue-and-table-storage.md) | 30 min | Python |
| 06 | [Storage Security](06-storage-security.md) | 60 min | CLI |
| 07 | [Azure NetApp Files and Elastic SAN](07-azure-netapp-files-and-elastic-san.md) | 35 min | – |
| 08 | [Azure Managed Lustre](08-azure-managed-lustre.md) | 25 min | – |
| 09 | [Storage Actions, Discovery and Insights](09-storage-actions-discovery-and-insights.md) | 30 min | Portal |
| 10 | [Data Transfer](10-data-transfer.md) | 40 min | AzCopy |
| 11 | [Storage for AI Workloads](11-storage-for-ai-workloads.md) | 45 min | CLI, Python |

**AI engineers:** 01, 02, 03, 06 and 11 in depth. **Architects:** all of it, with 01, 02, 06 and 11 in depth. **Consultants:** 01, 02, 04, 07 and 10 for positioning and migration conversations.

## Completion checklist

- [ ] I can explain LRS, ZRS, GRS, GZRS and their read-access variants, and state what each protects against.
- [ ] I can describe what happens during a customer-managed account failover.
- [ ] I have written a lifecycle policy that moves blobs to cool and archive and deletes old versions.
- [ ] I can explain the difference between versioning, soft delete, point-in-time restore and immutability.
- [ ] I can explain what hierarchical namespace changes and when to enable it.
- [ ] I can configure identity-based access for an Azure Files share.
- [ ] I have disabled shared key access on an account and accessed it with Entra ID.
- [ ] I can explain the three types of SAS and why user delegation SAS is preferred.
- [ ] I can pick a data transfer tool for 50 GB, 50 TB and 5 PB.
- [ ] I can sketch a storage layout for a RAG corpus with per-department access.

## Maps to

- **AZ-900**: storage services and redundancy.
- **AZ-104**: implement and manage storage.
- **AZ-305**: design data storage solutions.
- **AI-200**: develop for Azure Storage.
- **DP-700**: lakehouse storage concepts.

## Next

[09 · Databases](../09-databases/README.md)
