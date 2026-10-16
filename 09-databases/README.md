# 09 · Databases

> Tracks: E ● A ● C ○ · Part B · Core Infrastructure and Application Platform

Every application needs somewhere to keep its state, and AI applications need more of it than most: conversation history, agent memory, user profiles, product catalogues, embeddings and the operational data agents act on. Azure offers managed relational, document, key-value, graph and in-memory databases, and most now support vector search. This topic helps you choose between them, run them securely and reliably, and use them as the memory and state layer for AI agents.

## What you will be able to do

- Choose a database for a workload using a structured decision guide, and defend the choice.
- Size and configure Azure SQL Database, SQL Managed Instance, PostgreSQL and MySQL flexible servers.
- Model data in Azure Cosmos DB: partition keys, consistency levels, request units, global distribution.
- Position Azure DocumentDB and Azure Managed Redis, and plan migration from Azure Cache for Redis.
- Store and query vectors in Azure databases, and decide when to use Azure AI Search instead.
- Design conversation history, checkpoints and long-term memory for agents.
- Secure databases with Entra authentication, encryption, auditing and private networking.
- Design backup, high availability and disaster recovery to meet stated RPO and RTO.
- Plan a database migration with the right tools and cutover strategy.

## Prerequisites

[Topic 02](../02-identity-and-access/README.md) (Entra, managed identities), [Topic 05](../05-networking/README.md) (private endpoints, DNS) and [Topic 08](../08-storage/README.md) (redundancy concepts). Basic SQL and JSON are assumed.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Database Decision Guide](01-database-decision-guide.md) | 45 min | Exercise |
| 02 | [Azure SQL Database](02-azure-sql-database.md) | 60 min | CLI, SQL |
| 03 | [Azure SQL Managed Instance and SQL Server on VMs](03-azure-sql-managed-instance-and-sql-server-on-vms.md) | 45 min | – |
| 04 | [Azure Database for PostgreSQL](04-azure-database-for-postgresql.md) | 60 min | CLI, SQL |
| 05 | [Azure Database for MySQL](05-azure-database-for-mysql.md) | 30 min | CLI |
| 06 | [Azure Cosmos DB](06-azure-cosmos-db.md) | 75 min | CLI, Python |
| 07 | [Azure DocumentDB](07-azure-documentdb.md) | 30 min | CLI |
| 08 | [Azure Managed Redis](08-azure-managed-redis.md) | 45 min | CLI, Python |
| 09 | [Vector Search in Azure Databases](09-vector-search-in-azure-databases.md) | 60 min | SQL, Python |
| 10 | [Databases as Agent Memory and State](10-databases-as-agent-memory-and-state.md) | 50 min | Python |
| 11 | [Database Security and Compliance](11-database-security-and-compliance.md) | 50 min | CLI, SQL |
| 12 | [Backup, HA and DR for Databases](12-backup-ha-and-dr-for-databases.md) | 50 min | CLI |
| 13 | [Database Migration](13-database-migration.md) | 45 min | – |

**AI engineers:** 01, 04, 06, 08, 09 and 10 in depth. **Architects:** all of it, with 01, 06, 09, 11 and 12 in depth. **Consultants:** 01, 03, 07, 08 (migration) and 13.

## Completion checklist

- [ ] I can recommend a database for five different workloads and explain why.
- [ ] I can explain vCore vs DTU, serverless and Hyperscale in Azure SQL Database.
- [ ] I can explain when SQL Managed Instance is needed instead of SQL Database.
- [ ] I have enabled pgvector on PostgreSQL and run a similarity query.
- [ ] I can choose a Cosmos DB partition key and explain the consequences of a bad one.
- [ ] I can name the five Cosmos DB consistency levels and when to use each.
- [ ] I can explain the Azure Cache for Redis retirement and the migration target.
- [ ] I can explain when vectors belong in the operational database and when in Azure AI Search.
- [ ] I can design storage for agent threads, checkpoints and long-term memory.
- [ ] I have connected to a database with Entra authentication and no password.
- [ ] I can state RPO and RTO for geo-replication, failover groups and geo-restore.

## Maps to

- **AZ-900**: database services.
- **AZ-104**: limited (backup, networking).
- **AZ-305**: design data storage solutions (relational and non-relational).
- **AI-200**: develop solutions with Cosmos DB and relational databases.
- **DP-300**: Azure database administration (deeper SQL coverage).
- **AI-103**: state, memory and vector storage for AI apps.

## Next

[10 · Application Platform](../10-application-platform/README.md)
