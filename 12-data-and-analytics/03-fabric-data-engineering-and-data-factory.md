# Fabric Data Engineering and Data Factory

## Why it matters

Before data can feed reports, agents or RAG, someone has to ingest it, clean it and shape it. In Microsoft Fabric that work happens in two closely linked workloads. **Data Factory** does the ingestion and orchestration: pipelines, Copy jobs, Dataflows Gen2 and Airflow jobs. **Data Engineering** does the processing: lakehouses, Spark notebooks, Spark job definitions and environments. Architects need to know which tool fits which job. Consultants need to estimate how much capacity these workloads will consume.

## Key concepts

### The lakehouse item

A **lakehouse** is a Fabric item that combines files and tables in OneLake:

| Part | Description |
|---|---|
| **Tables** folder | Managed Delta tables, automatically discovered and registered in the metastore |
| **Files** folder | Unstructured or semi-structured files in any format (CSV, JSON, images, PDFs) |
| **SQL analytics endpoint** | A read-only T-SQL endpoint generated automatically over the Delta tables (views, functions and security can be defined; data changes are made with Spark or pipelines) |
| **Schemas** | Schema-enabled lakehouses organise tables into schemas (`dbo`, `sales`...) and support schema shortcuts |
| **Shortcuts** | Zero-copy links to other OneLake items or external storage (file 02) |

### Spark in Fabric

| Concept | Detail |
|---|---|
| **Fabric Runtime** | A versioned bundle of Apache Spark, Delta Lake, Python, and libraries (for example Runtime 1.3 ships Spark 3.5 and Delta 3.2; check the runtime page for newer runtimes and their status) |
| **Starter pools** | Pre-warmed Spark clusters that start in seconds, sized from the capacity |
| **Custom pools** | Your own node size, autoscale and dynamic executor allocation settings |
| **Environments** | Items that bundle Spark settings, a runtime version and libraries (public and custom), attached to notebooks and jobs |
| **High concurrency mode** | Several notebooks share one Spark session to save start-up time and CUs |
| **Native execution engine** | A vectorised C++ engine (based on Velox and Apache Gluten) that accelerates Spark SQL and DataFrame queries on Parquet and Delta |
| **V-Order** | A write-time optimisation of Parquet files for fast reads by all Fabric engines (especially Direct Lake and SQL) |
| **Optimize write / auto compaction** | Reduce small-file problems |
| **notebookutils** | Utilities (formerly `mssparkutils`) for files, credentials, notebook chaining and lakehouse access |

### Data Factory in Fabric

| Tool | What it is | Best for |
|---|---|---|
| **Pipelines** | Orchestration with activities (Copy, Notebook, Dataflow, Stored procedure, Script, Web, ForEach, If, Until, Invoke pipeline...) and schedules or event triggers | Orchestrating end-to-end loads, control flow, error handling |
| **Copy job** | A simplified item for bulk and **incremental** copy from many sources to many destinations, without building a pipeline | "Just move this data, and keep it up to date" |
| **Dataflows Gen2** | Power Query Online with 300+ transformations, writing to lakehouses, warehouses, SQL databases and other destinations; supports CI/CD and fast copy | Low-code shaping by analysts and citizen data engineers |
| **Apache Airflow job** | Managed Apache Airflow for Python-defined DAGs | Teams standardised on Airflow |
| **Mirroring** | Near real-time replicas of operational databases (file 02) | Replacing CDC pipelines |
| **Connectors and gateways** | 100+ connectors; **on-premises data gateway** and **virtual network data gateway** for private sources | Reaching sources behind firewalls |

Fabric pipelines are close relatives of Azure Data Factory pipelines but not identical. Datasets and linked services are replaced by **connections**, and some ADF features differ or aren't present. See file 09 for migration.

## How it works on Azure

### A medallion notebook (PySpark)

```python
from pyspark.sql import functions as F

# Bronze: raw JSON landed in Files/ by a pipeline or Copy job
raw = spark.read.json("Files/bronze/orders/2026/11/*.json")
raw.withColumn("_ingested_at", F.current_timestamp()) \
   .write.mode("append").format("delta").saveAsTable("bronze_orders")

# Silver: dedupe, type and validate
silver = (spark.table("bronze_orders")
          .dropDuplicates(["order_id"])
          .withColumn("order_date", F.to_date("order_date"))
          .filter(F.col("amount") >= 0))

from delta.tables import DeltaTable
if spark.catalog.tableExists("silver_orders"):
    (DeltaTable.forName(spark, "silver_orders").alias("t")
       .merge(silver.alias("s"), "t.order_id = s.order_id")
       .whenMatchedUpdateAll().whenNotMatchedInsertAll().execute())
else:
    silver.write.format("delta").saveAsTable("silver_orders")

# Gold: business aggregate for Power BI (Direct Lake)
(spark.table("silver_orders")
   .groupBy("region", F.date_trunc("month", "order_date").alias("month"))
   .agg(F.sum("amount").alias("revenue"), F.countDistinct("customer_id").alias("customers"))
   .write.mode("overwrite").format("delta").saveAsTable("gold_revenue_by_region_month"))
```

### Table maintenance

```sql
-- Spark SQL in a notebook
OPTIMIZE silver_orders VORDER;          -- compact small files and apply V-Order
VACUUM silver_orders RETAIN 168 HOURS;  -- remove unreferenced files older than 7 days
```

You can also run maintenance from the lakehouse UI or the REST API. Keep the VACUUM retention at least as long as any time travel or Direct Lake framing you depend on.

### Orchestration pattern

```text
Pipeline "daily_orders" (schedule 02:00, or event trigger on file arrival)
  ├─ Copy job / Copy activity: SQL Server (via gateway) → lakehouse Files/bronze
  ├─ Notebook: bronze → silver (parameters: run_date)
  ├─ Notebook: silver → gold
  ├─ Semantic model refresh (if not Direct Lake) 
  └─ On failure: Teams / Outlook activity or Activator alert
```

Parameterise notebooks (a cell tagged **parameters**) and pass values from pipeline activities. Use **variable libraries** and **deployment pipelines** or **Git integration** to move items between dev, test and prod.

### Dataflows Gen2 vs notebooks vs Copy

| Requirement | Choose |
|---|---|
| Analysts comfortable with Power Query, moderate volumes | Dataflows Gen2 |
| Large volumes, complex logic, unit tests, ML | Spark notebooks |
| Move data as-is (full or incremental) | Copy job or Copy activity |
| Operational database to analytics, near real time | Mirroring |
| Streaming events | Eventstreams (file 05) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Lakehouse vs Warehouse as the main store | Lakehouse for Spark-centric engineering and mixed data types; Warehouse for T-SQL-centric teams who need multi-table transactions and full DML. Both store Delta in OneLake and can query each other (file 04) |
| Notebooks vs Spark job definitions | Notebooks for interactive development and most production jobs; Spark job definitions for packaged code (JAR or Python files) built in CI |
| Starter vs custom pools | Starter pools for fast start-up; custom pools for specific node sizes or libraries (start-up time is longer) |
| Workspace per layer vs per domain | Per domain and environment is most common; separate bronze, silver and gold lakehouses (or schemas) inside it |
| ADF vs Fabric Data Factory | New Fabric-centric projects use Fabric Data Factory; existing ADF estates can stay or migrate gradually (file 09) |
| Capacity planning | Spark and pipeline runs are background operations (smoothed over 24 hours). Size the capacity on daily CU-hours, not peak minutes, and isolate heavy engineering from BI if needed |

## Hands-on

1. Create a schema-enabled lakehouse with `bronze`, `silver` and `gold` schemas.
2. Use a **Copy job** to load a public sample (or an Azure SQL table) into `bronze` incrementally.
3. Write the medallion notebook above, parameterise `run_date`, and run it from a pipeline with a failure notification.
4. Build a **Dataflow Gen2** that cleans a CSV and writes to `silver` in the same lakehouse; compare the CU usage of both approaches in the Capacity Metrics app.
5. Connect the workspace to a Git repository and commit the notebook, pipeline and lakehouse definitions.

## Security, cost and reliability notes

- **Security:** use workspace identity, managed private endpoints and trusted workspace access to reach firewalled sources. Keep secrets in **Azure Key Vault** and read them with `notebookutils.credentials.getSecret`.
- **Security:** restrict who can edit notebooks in production workspaces. A notebook runs with the identity of whoever runs or schedules it.
- **Cost:** idle interactive Spark sessions consume CUs until they time out. Set session timeouts and stop sessions when done. High concurrency mode reduces overhead.
- **Cost:** small files and un-optimised tables slow every downstream engine. Schedule maintenance.
- **Reliability:** make every notebook idempotent (MERGE, overwrite partitions), parameterised and restartable. Add retries on pipeline activities.
- **Reliability:** pin environment and runtime versions in production; test runtime upgrades in dev first.

## Common mistakes

- Writing many tiny Delta files from streaming or row-by-row inserts without compaction.
- Using the SQL analytics endpoint and expecting to run `INSERT` or `UPDATE` on lakehouse tables.
- Hard-coding workspace and lakehouse names instead of parameters and variable libraries.
- Running heavy Spark jobs on the same capacity as executive dashboards without monitoring throttling.
- Building transformations in Dataflows Gen2 at volumes better suited to Spark.
- Over-aggressive VACUUM that breaks readers.

## Check yourself

1. What is the difference between the Tables and Files sections of a lakehouse?
2. Can you update lakehouse tables through the SQL analytics endpoint?
3. What do environments provide?
4. When would you choose a Copy job over a pipeline?
5. What do V-Order and OPTIMIZE do?
6. How do you pass parameters from a pipeline to a notebook?
7. Why are Spark jobs less likely than report queries to cause throttling spikes?

<details>
<summary>Answers</summary>

1. Tables holds managed Delta tables registered automatically; Files holds files in any format and any folder structure.
2. No. It's read-only for table data; use Spark, pipelines or Dataflows to modify data.
3. A reusable bundle of Spark runtime version, Spark properties and libraries attached to notebooks and Spark job definitions.
4. When you simply need to copy data (full or incremental) from source to destination without custom orchestration logic.
5. V-Order optimises Parquet layout for fast reads across Fabric engines; OPTIMIZE compacts small files (and can apply V-Order).
6. Mark a notebook cell as a parameters cell and set base parameters in the pipeline's Notebook activity.
7. They're background operations, smoothed over 24 hours, while interactive operations are smoothed over a much shorter window.

</details>

## References

- [What is Data Engineering in Fabric?](https://learn.microsoft.com/fabric/data-engineering/data-engineering-overview)
- [Lakehouse overview](https://learn.microsoft.com/fabric/data-engineering/lakehouse-overview)
- [Apache Spark runtimes in Fabric](https://learn.microsoft.com/fabric/data-engineering/runtime)
- [Native execution engine](https://learn.microsoft.com/fabric/data-engineering/native-execution-engine-overview)
- [Delta Lake table optimisation and V-Order](https://learn.microsoft.com/fabric/data-engineering/delta-optimization-and-v-order)
- [What is Data Factory in Microsoft Fabric?](https://learn.microsoft.com/fabric/data-factory/data-factory-overview)
- [What is Copy job?](https://learn.microsoft.com/fabric/data-factory/what-is-copy-job)
- [Dataflow Gen2 overview](https://learn.microsoft.com/fabric/data-factory/dataflows-gen2-overview)
- [NotebookUtils](https://learn.microsoft.com/fabric/data-engineering/notebook-utilities)
