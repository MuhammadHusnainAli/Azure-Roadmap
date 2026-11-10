# Fabric Data Warehouse and Databases

## Why it matters

Fabric isn't just a lake. It has a full **T-SQL data warehouse** and, more recently, **operational databases**: **SQL database in Fabric** and **Cosmos DB in Fabric**. These run transactional applications inside the same SaaS platform, and their data is automatically replicated into OneLake for analytics. For architects this blurs the old OLTP/OLAP boundary. One platform can host an AI application's operational store (chat history, agent state, product catalogue with vectors) and the analytics on it, with no ETL in between.

## Key concepts

### The SQL-flavoured items in Fabric

| Item | Engine | Read/write | Purpose |
|---|---|---|---|
| **SQL analytics endpoint** (of a lakehouse or mirrored database) | Fabric Warehouse distributed engine | **Read-only** over Delta tables (plus views, functions, security) | Query lakehouse data with T-SQL |
| **Warehouse** | Fabric Warehouse distributed engine | **Full DML/DDL**, multi-table ACID transactions | Analytical warehouse for SQL-centric teams |
| **SQL database in Fabric** | Azure SQL Database engine (serverless, autonomous) | OLTP reads and writes | Operational relational apps; auto-mirrored to OneLake |
| **Cosmos DB in Fabric** | Azure Cosmos DB for NoSQL engine | Operational NoSQL reads and writes | Semi-structured, low-latency apps and AI; auto-mirrored to OneLake |

### Fabric Warehouse

| Feature | Detail |
|---|---|
| Storage | Delta Parquet in OneLake, readable by Spark, Power BI and other engines without copying |
| Compute | Serverless, distributed, autonomous: no indexes, distributions or vacuum to manage manually; automatic statistics and compaction |
| T-SQL surface | Tables, views, functions, stored procedures, `COPY INTO`, `CREATE TABLE AS SELECT` (CTAS), `MERGE` (check the current T-SQL surface area for limitations) |
| Transactions | Multi-statement, multi-table ACID transactions (snapshot isolation) |
| Cross-database queries | Three-part names join warehouses, lakehouse SQL endpoints and mirrored databases in the same workspace |
| **Zero-copy clone** | `CREATE TABLE ... AS CLONE OF ...` creates a metadata-only copy, optionally at a point in time |
| **Time travel** | `OPTION (FOR TIMESTAMP AS OF '...')` queries data as of a past point within retention |
| **Warehouse snapshots** | Read-only, point-in-time views of a whole warehouse for stable reporting during ETL |
| Security | Workspace roles, item permissions, `GRANT`/`DENY`, row-level security, column-level security, dynamic data masking |

### SQL database in Fabric

- Built on the **Azure SQL Database** engine and created as a Fabric item in seconds. There are no servers or tiers to choose, and it scales and pauses automatically.
- Data is **automatically mirrored** into OneLake as Delta in near real time, so analytics and Power BI never touch the OLTP engine.
- Supports normal app connectivity (TDS, ODBC, JDBC, ADO.NET) with **Microsoft Entra authentication only**.
- Includes modern Azure SQL features such as JSON, the native **vector** data type and vector functions (check the current feature comparison with Azure SQL Database).
- Integrates with Git and deployment pipelines (SQL projects) and with GraphQL APIs and user data functions in Fabric.
- Billed in Fabric CUs from the capacity, plus storage.

### Cosmos DB in Fabric

- Same engine as **Azure Cosmos DB for NoSQL**, with autonomous defaults and no throughput settings to tune, billed from Fabric capacity.
- Data is automatically surfaced in OneLake in Delta Parquet for analytics, cross-item queries and Power BI.
- Built-in **vector search**, **full-text search** and **hybrid search** (RRF), which suits AI apps, RAG over operational data and agent memory.
- Entra ID authentication for apps and users.

Status (checked 2026-11): Warehouse and SQL database in Fabric are generally available. Cosmos DB in Fabric became generally available in November 2025, alongside GA of Azure Cosmos DB mirroring. Individual features within each can still be in preview, so confirm regional availability on the Fabric release notes before committing.

## How it works on Azure

### Loading and transforming in the Warehouse

```sql
-- Ingest Parquet from ADLS with the warehouse's identity or a SAS (Entra preferred)
COPY INTO dbo.stg_sales
FROM 'https://stcontoso.dfs.core.windows.net/landing/sales/*.parquet'
WITH (FILE_TYPE = 'PARQUET');

-- Transform with CTAS
CREATE TABLE dbo.fact_sales AS
SELECT s.order_id, s.order_date, c.customer_key, s.amount
FROM dbo.stg_sales AS s
JOIN dbo.dim_customer AS c ON c.customer_id = s.customer_id;

-- Cross-database query: join warehouse data with a lakehouse table and a mirrored DB
SELECT f.order_id, l.segment, m.credit_limit
FROM dbo.fact_sales AS f
JOIN SalesLakehouse.dbo.customer_segments AS l ON l.customer_id = f.customer_key
JOIN MirroredCRM.dbo.accounts AS m ON m.account_id = f.customer_key;

-- Zero-copy clone for a test environment
CREATE TABLE dbo.fact_sales_test AS CLONE OF dbo.fact_sales;

-- Time travel
SELECT SUM(amount) FROM dbo.fact_sales OPTION (FOR TIMESTAMP AS OF '2026-11-01T00:00:00');
```

### Using SQL database in Fabric from an app (Python, keyless)

```python
import struct, pyodbc
from azure.identity import DefaultAzureCredential

token = DefaultAzureCredential().get_token("https://database.windows.net/.default").token
tok = token.encode("utf-16-le")
attrs = {1256: struct.pack(f"<I{len(tok)}s", len(tok), tok)}   # SQL_COPT_SS_ACCESS_TOKEN

conn = pyodbc.connect(
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=<server-from-connection-strings>,1433;Database=<db-name>;Encrypt=yes;",
    attrs_before=attrs)
conn.execute("INSERT INTO dbo.chat_messages(session_id, role, content) VALUES (?, ?, ?)",
             "s-42", "user", "Where is my order?")
conn.commit()
```

Copy the server and database names from the item's **Connection strings** page.

### Choosing where operational AI data lives

| Data | Good fit |
|---|---|
| Relational app data, transactions, reference data with vectors | SQL database in Fabric (or Azure SQL Database) |
| Chat history, agent memory, JSON documents, high-scale key-value with vectors | Cosmos DB in Fabric (or Azure Cosmos DB) |
| Curated analytical facts and dimensions | Warehouse or lakehouse gold tables |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Warehouse vs lakehouse | Warehouse for T-SQL teams, stored procedures, multi-table transactions and full DML; lakehouse for Spark, diverse files and data science. Many solutions use both on the same OneLake data |
| SQL database in Fabric vs Azure SQL Database | Fabric when the app is tightly coupled to Fabric analytics and you want zero-ETL and capacity billing. Azure SQL when you need full control of tiers, networking (private endpoints, VNet), geo-replication, the broadest feature set, or Azure-native billing |
| Cosmos DB in Fabric vs Azure Cosmos DB | Same reasoning: Fabric for integrated analytics and simplicity; Azure Cosmos DB for global distribution, multi-region writes, fine throughput control, other APIs (MongoDB, Cassandra, Gremlin, Table) and advanced networking |
| Synapse dedicated SQL pool migration | Fabric Warehouse is the target; plan T-SQL compatibility testing and use the migration assistant (file 09) |
| Shared vs separate capacity for OLTP | Operational databases consume the same capacity as analytics. Isolate on their own capacity when predictable latency matters |

## Hands-on

1. Create a Warehouse, load the sample data, and write a CTAS fact table.
2. Run a cross-database query joining the Warehouse with a lakehouse table in the same workspace.
3. Create a zero-copy clone and run a time-travel query.
4. Create a **SQL database in Fabric**, create a table with a `VECTOR` column, insert rows from Python using Entra authentication, and confirm the table appears (mirrored) in OneLake for a Spark notebook.
5. Create a **Cosmos DB in Fabric** database, insert a few chat-history documents, and query the mirrored data from the SQL analytics endpoint.

## Security, cost and reliability notes

- **Security:** use Entra ID everywhere. SQL database in Fabric doesn't support SQL authentication.
- **Security:** apply RLS, CLS and masking in the Warehouse. Remember that Spark and OneLake access to the same Delta files are governed separately (OneLake security, item permissions).
- **Cost:** everything draws from capacity CUs. Warehouse queries are mostly classified as background (24-hour smoothing); operational databases add continuous load, so monitor them in the Capacity Metrics app.
- **Reliability:** Warehouse supports restore points (system and user-defined) for in-place restore. SQL database in Fabric has automatic backups with point-in-time restore. Check retention periods and cross-region options against your RPO and RTO.
- **Reliability:** operational SLAs for Fabric databases differ from Azure PaaS equivalents. Validate them for mission-critical apps.

## Common mistakes

- Expecting the lakehouse SQL analytics endpoint to accept writes.
- Running a latency-sensitive OLTP app on a capacity also running heavy Spark jobs.
- Porting Synapse dedicated pool code with distributions and index hints unchanged.
- Choosing SQL database in Fabric when the requirement includes private endpoints or geo-replication that only Azure SQL provides.
- Forgetting that mirrored data in OneLake is visible to anyone with access to it there.

## Check yourself

1. What can you do in a Warehouse that you can't do through a lakehouse SQL analytics endpoint?
2. What is a zero-copy clone?
3. How does data from SQL database in Fabric reach analytics engines?
4. When would you choose Azure Cosmos DB over Cosmos DB in Fabric?
5. Which authentication methods does SQL database in Fabric support?
6. How do you query data across a warehouse and a mirrored database?

<details>
<summary>Answers</summary>

1. Full DML and DDL on tables (INSERT, UPDATE, DELETE, MERGE, CTAS) with multi-table transactions; the SQL analytics endpoint is read-only for table data.
2. A metadata-only copy of a table (optionally at a past point in time) that shares the underlying files, created instantly without duplicating data.
3. It's automatically mirrored into OneLake as Delta tables in near real time, available to Spark, SQL and Power BI.
4. When you need global distribution, multi-region writes, fine-grained throughput control, other APIs, advanced networking, or Azure-native billing.
5. Microsoft Entra ID only.
6. Use three-part names (`database.schema.table`) in a T-SQL query from the Warehouse or SQL endpoint in the same workspace.

</details>

## References

- [What is data warehousing in Microsoft Fabric?](https://learn.microsoft.com/fabric/data-warehouse/data-warehousing)
- [Warehouse vs lakehouse SQL analytics endpoint](https://learn.microsoft.com/fabric/data-warehouse/data-warehousing#warehouse-or-lakehouse)
- [Zero-copy clone](https://learn.microsoft.com/fabric/data-warehouse/clone-table)
- [Time travel in Warehouse](https://learn.microsoft.com/fabric/data-warehouse/time-travel)
- [SQL database in Microsoft Fabric](https://learn.microsoft.com/fabric/database/sql/overview)
- [Cosmos DB in Microsoft Fabric](https://learn.microsoft.com/fabric/database/cosmos-db/overview)
- [T-SQL surface area in Fabric Data Warehouse](https://learn.microsoft.com/fabric/data-warehouse/tsql-surface-area)
