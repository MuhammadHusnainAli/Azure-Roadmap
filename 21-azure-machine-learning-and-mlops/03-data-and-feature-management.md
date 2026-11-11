# Data and Feature Management

## Why it matters

Models are only as good as the data they're trained on, and only reproducible if you know exactly which data version produced them. Most production ML failures (training/serving skew, data leakage, "it worked last month") are data management failures. Azure ML gives you **datastores** for secure connections, versioned **data assets** for reproducibility, and a **managed feature store** for reusable, consistent features across training and inference.

## Key concepts

### Datastores

A **datastore** is a reference to an existing Azure storage service with connection details held by the workspace, so scripts don't contain credentials.

| Storage | Credential-based (key, SAS, service principal) | Identity-based (Entra user or managed identity) |
|---|---|---|
| Blob container | ✓ | ✓ |
| ADLS Gen2 | ✓ | ✓ |
| Azure Files | ✓ | — |
| ADLS Gen1 (legacy) | ✓ | ✓ |

Default datastores: `workspaceblobstore` (uploads, code snapshots, pipeline cache) and `workspaceworkingdirectory` (file share for notebooks). Credential-based datastores expose secrets to users with workspace Reader access to credentials; **prefer identity-based** access. Synapse SQL isn't supported as a data import source.

### Data assets

| Type | Represents | Use |
|---|---|---|
| `uri_file` | A single file | A CSV or Parquet file |
| `uri_folder` | A folder | Images, many files |
| `mltable` | A tabular abstraction with a schema and load steps (`MLTable` file) | Tables that need consistent parsing, filtering, column types |

Data assets are **versioned** and **immutable per version**, referenced in jobs as `azureml:churn-data:3`. Jobs access data by **mount** (stream on demand) or **download** (copy to local disk), and write outputs with `rw_mount` or `upload`.

Other data features: **data import** (scheduled import from external sources such as Snowflake or Azure SQL into a data asset), **data labelling** projects (images, text), and lineage between data assets, jobs and models.

### Managed feature store

A **feature** is model input data, often derived through aggregates and transformations (for example "customer's transaction count in the last 30 days").

| Problem | How the managed feature store helps |
|---|---|
| Teams rebuild the same features | Search and reuse features across stores and teams |
| Feature engineering pipelines are hard to operate | Declarative **feature set specifications** with transformation code; managed **materialisation** on managed Spark |
| Training/serving skew | One definition used for training data generation and inference |
| Data leakage in training sets | **Point-in-time joins** ("time travel") when retrieving features |
| Breaking older models | Feature sets are **versioned and immutable** |

Components:

| Component | Description |
|---|---|
| **Feature store** | A workspace type shared by many project workspaces (also usable from Databricks and local Spark) |
| **Entity** | The key features describe (customer, account) |
| **Feature set** | Versioned set of features with source, transformation and materialisation settings |
| **Materialisation store** | **Offline** (ADLS Gen2) for training and batch; **online** (Redis) for low-latency inference |
| **Feature retrieval spec** | The list of features a model uses; travels with the model for consistent inference |
| **Retrieval** | Built-in pipeline component or `get_offline_features()` for training and batch data |

Monitoring covers materialisation job status and notifications; RBAC applies to stores, feature sets and entities. The main cost is managed Spark materialisation.

## How it works on Azure

Identity-based ADLS datastore and a versioned data asset:

```yaml
# datastore.yml
$schema: https://azuremlschemas.azureedge.net/latest/azureDataLakeGen2.schema.json
name: lake_curated
type: azure_data_lake_gen2
account_name: contosolake
filesystem: curated
# no credentials → identity-based access
```

```yaml
# data.yml
$schema: https://azuremlschemas.azureedge.net/latest/data.schema.json
name: churn-training
version: 3
type: uri_folder
path: azureml://datastores/lake_curated/paths/churn/2026-10/
```

```bash
az ml datastore create -f datastore.yml
az ml data create -f data.yml
```

Use in a job:

```yaml
inputs:
  training_data:
    type: uri_folder
    path: azureml:churn-training:3
    mode: ro_mount
```

Grant the compute's managed identity (and users) **Storage Blob Data Reader** on the lake for identity-based access.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Identity vs credential datastores** | Identity-based by default; credential-based only when identities can't be used. |
| **uri_folder vs mltable** | `uri_folder`/`uri_file` for files consumed by your own code; `mltable` when many jobs need the same parsing and schema. |
| **Mount vs download** | Mount for large datasets read partially; download for small data read many times. |
| **Feature store or not** | Worth it when multiple models and teams reuse features, online inference needs low-latency features, or skew/leakage has bitten you. Overkill for a single model. |
| **Feature store vs Databricks/Fabric** | If your ML platform is Databricks, its feature engineering features may fit better; Azure ML's store can still be consumed from Databricks. |
| **Versioning discipline** | Never overwrite data behind a version; create a new version and record it in model metadata. |

## Hands-on

1. Create an identity-based ADLS datastore and register two versions of a `uri_folder` data asset.
2. Run a training job against version 1, then version 2; compare lineage in studio (data asset → job → model).
3. Create an `MLTable` for a CSV with column types and filters; use it in a job.
4. Optional: follow the managed feature store sample to create a feature store, a `transactions` feature set with a 7-day window aggregate, materialise it offline, and generate a point-in-time training set.

## Security, cost and reliability notes

- **Least privilege** on storage for compute identities; private endpoints for lakes.
- **Personal data**: catalogue sensitive columns (Purview), minimise features, and respect retention in feature stores.
- **Cost**: storage, data import jobs, feature materialisation (managed Spark) and Redis for online features.
- **Reliability**: immutable versions make training reproducible; materialisation monitoring catches stale features.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hard-coding storage keys in notebooks | Identity-based datastores |
| Overwriting files behind a data asset version | Create new versions |
| Computing features differently in training and serving | Shared feature definitions (feature store or shared code) |
| Random train/test splits on time-dependent data | Point-in-time joins and time-based splits |

## Check yourself

1. What is a datastore, and which authentication options exist?
2. Name the three data asset types.
3. What's the difference between mount and download?
4. What problems does a managed feature store solve?
5. What are offline and online materialisation stores?
6. What are point-in-time joins for?

<details>
<summary>Answers</summary>

1. A reference to an existing storage service with connection info; credential-based (key, SAS, service principal) or identity-based (Entra/managed identity).
2. `uri_file`, `uri_folder`, `mltable`.
3. Mount streams data on demand; download copies it to local disk first.
4. Feature reuse, operating feature pipelines, training/serving skew, data leakage, versioning without breaking older models.
5. Offline: ADLS Gen2 for training and batch; online: Redis for low-latency inference.
6. Retrieving feature values as they were at each event's time, avoiding leakage of future data into training.

</details>

## References

- [Data concepts in Azure ML](https://learn.microsoft.com/azure/machine-learning/concept-data)
- [Create datastores](https://learn.microsoft.com/azure/machine-learning/how-to-datastore)
- [What is managed feature store?](https://learn.microsoft.com/azure/machine-learning/concept-what-is-managed-feature-store)
- [Working with tables (MLTable)](https://learn.microsoft.com/azure/machine-learning/how-to-mltable)
