# Healthcare Data Pipeline on Azure Databricks

Moves the client's Hospital and Patient data from an on-prem SQL Server into Azure, cleans it in Databricks, and keeps it current in Gold tables for Power BI reporting.

![Architecture](../architecture/project2_architecture.png)

## Architecture

```
On-prem SQL Server ──CSV──► ADLS Gen2 (raw/) ──► Azure Databricks ──► Power BI
 (Hospital, Patient)        hospital_raw.csv     Bronze → Silver → Gold
                            patient_raw.csv            ▲
                                                       │ secrets
                                                 Azure Key Vault
```

| Layer  | Location                         | Content                                   |
|--------|----------------------------------|-------------------------------------------|
| Raw    | `abfss://raw@adlsproject2anupa.dfs.core.windows.net/` | CSV exports from SQL Server |
| Bronze | DataFrames in `nb_adls_connection` | Raw data as loaded, no changes          |
| Silver | `silver.hospital`, `silver.patient` | Cleaned, typed, deduplicated Delta tables |
| Gold   | `gold.dim_hospital` (SCD 1), `gold.dim_patient` (SCD 2) | Report-ready dimensions |

## Tech stack

SQL Server · Azure Data Lake Storage Gen2 · Azure Key Vault · Azure Databricks (PySpark, Delta Lake, Unity Catalog) · Power BI

## Azure resources

| Resource | Name |
|----------|------|
| Storage account (ADLS Gen2) | `adlsproject2anupa`, container `raw` |
| Databricks workspace (Premium) | `dbw-project2-anupa` |
| Access Connector | `unity-catalog-access-connector` |
| Storage credential | `cred_adls_anupa` |
| External location | `ext_raw` |
| Key Vault | `kv-healthcare-anupa79`, secret `sas-adls-raw` |
| Databricks secret scope | `kv-scope` (Key Vault-backed) |

## Connecting Databricks to ADLS

1. **External location (Unity Catalog), used by the pipeline.** The Access Connector (managed identity) has *Storage Blob Data Contributor* on the storage account. The storage credential `cred_adls_anupa` wraps it, and the external location `ext_raw` points to the `raw` container. No keys appear in code.
2. **SAS key via Key Vault.** A read/list-only SAS token is stored in Key Vault as `sas-adls-raw` and read through the `kv-scope` secret scope in `nb_sas_connection`. This needs a dedicated cluster because `spark.conf` is not allowed on Serverless.

| Method | Pros | Cons |
|--------|------|------|
| External location | No secrets; central permissions and audit; works on Serverless | More setup; needs Premium and admin rights |
| SAS key | Limited rights; auto-expires | Token holder has access; must be renewed |
| Account key | Simplest | Full account access; no expiry; least secure |
| Service principal | Fine-grained RBAC; standard for automation | App registration; client secret rotation |

## Notebooks

| Notebook | Purpose |
|----------|---------|
| `nb_adls_connection` | Reads `hospital_raw.csv` and `patient_raw.csv` through `ext_raw` (Bronze) |
| `nb_sas_connection` | Alternative connection with the SAS token from Key Vault |
| `nb_transform` | `%run ./nb_adls_connection`, cleans the data and writes the Silver tables |
| `nb_sales` | Sales DataFrame: revenue, profit, highest cost per product, top customer, saved to `default.sales` |
| `nb_gold` | SCD Type 1 MERGE for `gold.dim_hospital`, SCD Type 2 for `gold.dim_patient` |

## Data cleaning (Silver)

- Drop rows with a null key (`na.drop(subset=[...])`)
- Trim text columns; `try_cast` numeric columns
- Parse dates in two formats with `coalesce(to_date(...), to_date(...))`
- `fillna` defaults (`"Unknown"`, `0`)
- `dropDuplicates` on the business key
- Rename columns (`bed_count` → `total_beds`, `dob` → `date_of_birth`)
- Add `load_ts = current_timestamp()`

## Slowly changing dimensions (Gold)

- **Hospital, SCD Type 1:** `MERGE` on `hospital_id` updates matched rows and inserts new ones. No history is kept. In testing, H001 went from 850 to 900 beds and H006 was inserted (6 rows).
- **Patient, SCD Type 2:** extra columns are `start_date`, `end_date` and `is_current`. A change closes the current row (`end_date = today`, `is_current = false`) and appends a new current row. In testing, P001 moved Mumbai → Pune and is kept as two rows, and P006 was inserted (7 rows).

## How to run

1. Export Hospital and Patient from SQL Server to CSV and upload them to `raw/`.
2. Run `nb_transform`, which also runs the connection notebook.
3. Run `nb_sales`.
4. Run `nb_gold`.
5. In Power BI Desktop, connect to the Serverless SQL warehouse (Azure Databricks connector, hostname + HTTP path, Personal Access Token, Import mode) and load `gold.dim_hospital` and `gold.dim_patient`.

## Reporting (Power BI)

- Column chart: `total_beds` by `hospital_name` (`gold.dim_hospital`)
- Table: `gold.dim_patient` with `start_date`, `end_date`, `is_current`, which shows the SCD Type 2 history

## Known limitations

- The SAS method needs a dedicated cluster. That cluster is blocked by the 4-vCPU regional quota in Central India.
- Files are loaded manually; there is no orchestration tool in the current scope.

## Documentation

Step-by-step setup with screenshots: `Healthcare_Pipeline_Documentation.docx`
