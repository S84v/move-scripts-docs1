**File:** `mediation-ddls\mnaas_activations_raw_daily.hql`  

---

### 1. High‑Level Summary
This script creates an **external Hive/Impala table** named `mnaas.activations_raw_daily`. The table stores daily activation records for MVNO (mobile virtual network operator) customers in Parquet format, partitioned by a string `partition_date`. Because it is external and has `external.table.purge='true'`, dropping the table will also delete the underlying files. The definition is used by downstream mediation jobs that aggregate, enrich, or report on activation data.

---

### 2. Core Objects Defined
| Object | Responsibility |
|--------|-----------------|
| **`CREATE EXTERNAL TABLE mnaas.activations_raw_daily`** | Registers the schema and storage location for raw activation files. |
| **Columns (≈ 45 fields)** | Capture business‑level identifiers (company, invoice, MVNO), product details (commercial offer, SIM/ICCID, status dates), technical metadata (filename, inserttime), and enrichment fields (carrier, rate‑plan, zone lists, etc.). |
| **`PARTITIONED BY (partition_date string)`** | Enables daily partition pruning; each partition corresponds to a processing date. |
| **`ROW FORMAT SERDE … ParquetHiveSerDe`** | Declares Parquet as the serialization format. |
| **`LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/activations_raw_daily'`** | Physical HDFS directory where raw files are landed. |
| **`TBLPROPERTIES`** | Controls statistics, purge behavior, Impala catalog IDs, and Spark schema metadata. |

*No procedural code, classes, or functions are present – the file is pure DDL.*

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - Parquet files placed (by upstream ingestion pipelines) under the HDFS location above, named per day (e.g., `.../partition_date=2024-12-03/`). <br> - Implicitly depends on the Hive metastore service and the HDFS namenode `NN-HA1`. |
| **Outputs** | - A Hive/Impala table metadata entry (`mnaas.activations_raw_daily`). <br> - Queryable data for downstream jobs (e.g., aggregation, KPI generation, billing). |
| **Side Effects** | - Registers the table in the Hive metastore. <br> - Because of `external.table.purge='true'`, a `DROP TABLE` will delete the underlying HDFS directory. |
| **Assumptions** | - Data files are already in Parquet and conform to the defined schema. <br> - The `partition_date` string matches the folder naming convention used by the ingestion pipeline. <br> - The Hive/Impala services have read/write access to the HDFS path. |

---

### 4. Integration Points  

| Connected Component | How It Links |
|---------------------|--------------|
| **Upstream ingestion jobs** (e.g., Spark or Flume jobs that land activation files) | Write Parquet files to the HDFS location; they must respect the column order and data types defined here. |
| **Downstream mediation scripts** (e.g., `mnaas_activations_daily_agg.hql`, `mnaas_activations_daily_report.hql`) | Reference `mnaas.activations_raw_daily` for aggregation, filtering by `partition_date`, and joining with other dimension tables (e.g., `dim_date`). |
| **Data catalog / governance tools** | Pull table metadata from Hive metastore; the `impala.events.catalogServiceId` and version properties are used for Impala cache invalidation. |
| **Scheduler / workflow engine** (e.g., Oozie, Airflow) | Executes this DDL as a one‑time “bootstrap” step or as part of a schema‑migration DAG. |
| **Environment configuration** | The HDFS namenode address (`NN-HA1`) and Hive warehouse root are typically injected via environment variables or a central `hive-site.xml`. |

---

### 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Schema drift** – upstream jobs produce columns that differ (type or order) from the DDL. | Enforce schema validation in the ingestion pipeline; version‑control the DDL and run `DESCRIBE FORMATTED` checks after each load. |
| **Partition mis‑alignment** – missing or incorrectly named `partition_date` folders cause queries to return empty sets. | Automate partition addition (`ALTER TABLE … ADD IF NOT EXISTS PARTITION`) as part of the load job; monitor HDFS directory structure. |
| **Accidental table drop** – `external.table.purge=true` will delete raw files. | Restrict DROP privileges; add a pre‑drop safeguard script that backs up the HDFS path or disables the purge flag temporarily. |
| **Permission changes** – Hive/Impala lose read/write rights to the HDFS location. | Periodically verify ACLs on the directory; embed a health‑check step in the workflow that runs `hdfs dfs -test -e …`. |
| **Namenode address change** – hard‑coded `NN-HA1` becomes stale after HA re‑configuration. | Externalize the namenode URI into a config file or environment variable (e.g., `HDFS_NN_URI`). |

---

### 6. Running / Debugging the Script  

1. **Execute** (once) from a Hive or Impala client:  
   ```bash
   hive -f mediation-ddls/mnaas_activations_raw_daily.hql
   # or
   impala-shell -i <impala-coordinator> -f mediation-ddls/mnaas_activations_raw_daily.hql
   ```
2. **Verify creation**:  
   ```sql
   SHOW CREATE TABLE mnaas.activations_raw_daily;
   DESCRIBE FORMATTED mnaas.activations_raw_daily;
   ```
3. **Check partitions** (after data load):  
   ```sql
   SHOW PARTITIONS mnaas.activations_raw_daily;
   ```
4. **Debug common issues**:  
   - *“Table already exists”* → add `IF NOT EXISTS` to the DDL.  
   - *“Permission denied”* → inspect HDFS ACLs (`hdfs dfs -ls <path>`).  
   - *“Invalid Parquet schema”* → compare the Parquet file schema (`parquet-tools schema <file>`) with the DDL column list.  

---

### 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` clause; should be sourced from a cluster‑wide config (`hdfs-site.xml` or env var). |
| Hive Metastore connection | Defined in `hive-site.xml`; required for table registration. |
| Impala catalog IDs (`impala.events.catalogServiceId`, `catalogVersion`) | Populated automatically; not manually edited but may be referenced by monitoring tools. |
| Optional: `HIVE_CONF_DIR`, `HADOOP_CONF_DIR` | Provide runtime configuration for Hive/Impala clients. |

---

### 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend `CREATE EXTERNAL TABLE IF NOT EXISTS` and wrap the `LOCATION` path in a variable so the script can be safely re‑run during CI/CD deployments.  
2. **Add a partition‑management helper** – a small Bash/Python snippet that scans the HDFS directory for new `partition_date` folders and issues `ALTER TABLE … ADD IF NOT EXISTS PARTITION` automatically, preventing “missing partition” query failures.  

---