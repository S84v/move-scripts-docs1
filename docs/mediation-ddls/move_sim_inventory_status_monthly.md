**File:** `mediation-ddls\move_sim_inventory_status_monthly.hql`  

---

### 1. High‑Level Summary
This script defines a **Hive external table** named `mnaas.move_sim_inventory_status_monthly`. The table stores a wide set of SIM‑inventory attributes (IMSI, MSISDN, SIM identifiers, status dates, device metadata, billing information, etc.) and is **partitioned by a string column `partition_month`**. The table points to a fixed HDFS location, uses a lazy CSV‑style SerDe, and is configured with Impala‑specific catalog properties and a purge‑on‑drop flag. In production the table is the landing zone for monthly SIM‑inventory status extracts that downstream jobs (e.g., reporting, reconciliation, or further ETL) query.

---

### 2. Core Object(s) & Responsibilities
| Object | Responsibility |
|--------|-----------------|
| **`CREATE EXTERNAL TABLE mnaas.move_sim_inventory_status_monthly`** | Registers a Hive/Impala external table with a predefined schema, partitioning, storage format, and location. |
| **`PARTITIONED BY (partition_month string)`** | Enables month‑level data isolation and incremental loading. |
| **`ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'`** | Parses delimited text files (default delimiter is `\001`). |
| **`STORED AS INPUTFORMAT/OUTPUTFORMAT`** | Uses TextInputFormat and HiveIgnoreKeyTextOutputFormat for plain‑text files. |
| **`TBLPROPERTIES`** | Controls statistics handling, purge behavior, and Impala catalog integration. |

*No procedural code, functions, or classes are present – the file is a pure DDL statement.*

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Raw data files (CSV‑like) placed under `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/move_sim_inventory_status_monthly`.<br>• Hive metastore connection (default for the environment). |
| **Outputs** | • Hive/Impala metadata entry for the external table.<br>• No data movement – the table simply references existing files. |
| **Side‑Effects** | • Registers the table in the metastore; subsequent queries can read the data.<br>• `external.table.purge='true'` causes underlying HDFS files to be **deleted** automatically if the table is dropped. |
| **Assumptions** | • Files in the location conform to the column order and data types defined.<br>• Partition directories follow the pattern `partition_month=YYYYMM` (or similar) so that Hive can discover partitions automatically.<br>• The HDFS namenode alias `NN-HA1` resolves correctly in the execution environment.<br>• Impala catalog IDs (`catalogServiceId`, `catalogVersion`) are kept in sync with the Hive metastore. |

---

### 4. Integration Points  

| Connected Component | How the Connection Happens |
|---------------------|-----------------------------|
| **`move_sim_inventory_status` (non‑monthly version)** | Likely loads daily or raw SIM status data; this monthly table may be populated by a downstream aggregation job that inserts into the partitioned table. |
| **ETL orchestration (e.g., Oozie, Airflow, Azkaban)** | A scheduled workflow runs a **INSERT OVERWRITE** or **LOAD DATA INPATH** job that writes month‑specific files into the HDFS location, then runs `MSCK REPAIR TABLE` or `ALTER TABLE … ADD PARTITION` to make them visible. |
| **Reporting / BI tools (Impala, Tableau, PowerBI)** | Query the table directly for month‑level SIM inventory dashboards. |
| **Data quality / validation scripts** | May read the table schema via `DESCRIBE FORMATTED` to validate incoming file structures before loading. |
| **Retention / cleanup jobs** | Use the `external.table.purge` flag to safely drop old partitions/tables without orphaned files. |

*Because the script only creates the table, any data‑load or transformation logic resides in separate `.hql` or Spark/MapReduce jobs that reference this table.*

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Schema drift** – upstream data files change column order or type, causing query failures. | Implement a schema‑validation step (e.g., a Spark job that checks column count/types before writing to the HDFS location). |
| **Stale partitions** – missing `MSCK REPAIR TABLE` after new month directories are added, leading to invisible data. | Automate `MSCK REPAIR TABLE mnaas.move_sim_inventory_status_monthly` in the load workflow or use `ALTER TABLE … ADD PARTITION`. |
| **Uncontrolled storage growth** – external table points to a location that is never cleaned. | Schedule periodic HDFS cleanup based on `partition_month` age; consider archiving old partitions to cheaper storage. |
| **Accidental table drop** – `external.table.purge=true` will delete all underlying files. | Restrict DROP privileges to a limited admin group; add a pre‑drop confirmation step in orchestration. |
| **Impala catalog mismatch** – catalog IDs become out‑of‑sync after metastore upgrades. | Include a health‑check that runs `SHOW TABLE STATS` via Impala and verifies `impala.lastComputeStatsTime`. |
| **Network / namenode alias failure** – `NN-HA1` may not resolve in all environments. | Use a configuration property (e.g., `${hdfs.namenode}`) that can be overridden per environment. |

---

### 6. Running / Debugging the Script  

| Action | Command / Steps |
|--------|-----------------|
| **Create / replace the table** | `hive -f move_sim_inventory_status_monthly.hql`  (or `beeline -u jdbc:hive2://<host>:10000 -f …`) |
| **Verify table definition** | `DESCRIBE FORMATTED mnaas.move_sim_inventory_status_monthly;` |
| **Check partitions** | `SHOW PARTITIONS mnaas.move_sim_inventory_status_monthly;` |
| **Refresh Impala metadata** (if using Impala) | `impala-shell -q "INVALIDATE METADATA mnaas.move_sim_inventory_status_monthly;"` |
| **Inspect underlying files** | `hdfs dfs -ls /user/hive/warehouse/mnaas.db/move_sim_inventory_status_monthly/partition_month=202312/` |
| **Debug missing data** | 1. Verify file presence and permissions.<br>2. Run `MSCK REPAIR TABLE …` to force partition discovery.<br>3. Check Hive logs (`/var/log/hive/hive.log`) for parsing errors. |

---

### 7. External Configurations & Environment Variables  

| Config / Variable | Usage |
|-------------------|-------|
| `NN-HA1` (HDFS namenode alias) | Hard‑coded in the `LOCATION` clause; should resolve via the cluster’s DNS or `/etc/hosts`. |
| Hive Metastore connection properties (e.g., `hive.metastore.uris`) | Required for the `CREATE EXTERNAL TABLE` command to succeed. |
| Impala catalog IDs (`impala.events.catalogServiceId`, `catalogVersion`) | Populated automatically by Impala; not manually edited but may be referenced by monitoring scripts. |
| Optional: `HIVE_CONF_DIR`, `HADOOP_CONF_DIR` | Provide Hadoop/Hive configuration files for the client executing the script. |

*If the environment uses parameterised locations (e.g., `${hdfs.base}`), consider refactoring the script to use a variable substitution mechanism rather than a hard‑coded URI.*

---

### 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard** – Prevents accidental failures when the table already exists:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.move_sim_inventory_status_monthly ( … )
   ```
2. **Externalize the HDFS location** – Replace the hard‑coded URI with a configurable variable (e.g., `${move_sim_inventory_status_monthly_path}`) sourced from a central properties file or environment variable, enabling easier migration across clusters.  

---