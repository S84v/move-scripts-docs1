**File:** `mediation-ddls/bl_cust_bu_co_cogrp_mapping.hql`  

---

### 1. Production Summary
This Hive DDL script creates (or replaces) an **external table** `mnaas.bl_cust_bu_co_cogrp_mapping` that stores the mapping between a customer’s organizational hierarchy (org‑id/name) and the contract‑offer‑group (COG) information used by the mediation layer. The table is defined over a delimited text file located in HDFS under the `mnaas` database warehouse. Because it is external, the data files are managed outside Hive, allowing downstream ETL jobs to drop/re‑create the table without affecting the underlying raw files.

---

### 2. Key Artefacts & Responsibilities  

| Artefact | Type | Responsibility |
|----------|------|----------------|
| `createtab_stmt` | Variable (string) | Holds the full `CREATE EXTERNAL TABLE` statement; the script may be sourced by a wrapper that executes the variable via Hive CLI or Beeline. |
| `mnaas.bl_cust_bu_co_cogrp_mapping` | Hive external table | Persists mapping rows with fields such as `org_id`, `org_name`, `contract_offer_group`, `commercial_offer`, various aggregation levels, and effective dates. |
| SerDe `LazySimpleSerDe` | Hive SerDe | Parses the source files using `~` as field delimiter and `\n` as line delimiter. |
| HDFS location `hdfs://NN-HA1/.../bl_cust_bu_co_cogrp_mapping` | Storage | Physical location of the raw delimited files that feed the table. |

*No procedural code (functions, classes) is present in this file; its sole purpose is schema definition.*

---

### 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - Raw data files placed (or staged) in the HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/bl_cust_bu_co_cogrp_mapping`.<br>- Implicit Hive/Impala configuration (e.g., `hive.metastore.uris`, `hdfs.namenode`). |
| **Outputs** | - Hive metastore entry for the external table (metadata only).<br>- No data is written by this script; the underlying files remain untouched. |
| **Side‑effects** | - Registers the table in the metastore, making it visible to Hive, Impala, and any downstream Spark/Presto jobs.<br>- Because `external.table.purge='true'`, dropping the table will also delete the underlying HDFS files (a destructive side‑effect). |
| **Assumptions** | - The HDFS path exists and is readable/writable by the Hive service user (`root` in the current metadata).<br>- Files conform to the `~`‑delimited format and contain exactly the columns defined (order matters).<br>- The cluster’s Impala catalog is synchronized with Hive metastore (see `impala.events.*` properties). |

---

### 4. Integration Points  

| Connected Component | Interaction |
|---------------------|-------------|
| **Ingestion jobs** (e.g., nightly SFTP pull, Kafka consumer, or batch file copy) | Place raw `~`‑delimited files into the HDFS location before this DDL is executed. |
| **Downstream ETL scripts** (SQL/DDL files that reference `bl_cust_bu_co_cogrp_mapping`) | Perform joins to enrich billing, rating, or reporting data. Typical usage: `SELECT ... FROM mnaas.bl_cust_bu_co_cogrp_mapping WHERE org_id = …`. |
| **Orchestration layer** (Airflow, Oozie, Control-M) | Executes this DDL as a task, often preceded by a “clean‑up” step that removes old files or truncates the directory. |
| **Impala** | Reads the table directly for low‑latency analytics; the `impala.events.*` properties ensure catalog updates propagate. |
| **Monitoring/Alerting** | Metastore change listeners or custom scripts may watch for `last_modified_time` to detect unexpected table recreation. |

*Because the table is external, any process that deletes or moves the HDFS directory will break downstream queries.*

---

### 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss** – `external.table.purge='true'` causes HDFS files to be removed when the table is dropped. | Permanent loss of raw mapping files. | Enforce a change‑control policy that forbids `DROP TABLE` without a prior backup of the directory; consider setting the property to `false` in non‑prod environments. |
| **Schema drift** – Upstream data producers change column order or delimiter. | Queries fail or return corrupted results. | Add a validation step (e.g., a Hive `SELECT COUNT(*) FROM ... LIMIT 1`) that checks column count and delimiter before the table is used. |
| **Permission issues** – Hive service user lacks HDFS read/write rights. | Table creation fails; downstream jobs cannot read data. | Verify HDFS ACLs (`hdfs dfs -ls -R …`) and ensure the Hive principal is granted `rwx` on the directory. |
| **Stale metadata** – Impala catalog not refreshed after DDL execution. | Queries see old schema or missing rows. | Include an Impala `INVALIDATE METADATA` or `REFRESH` command in the orchestration after the DDL runs. |
| **Large file ingestion** – Single massive file can cause mapper overload. | Job timeouts, OOM errors. | Encourage upstream producers to split files into manageable chunks (e.g., < 1 GB) and consider adding partitioning in a future redesign. |

---

### 6. Running / Debugging the Script  

1. **Typical execution (operator)**  
   ```bash
   # From the mediation‑ddls directory
   hive -f mediation-ddls/bl_cust_bu_co_cogrp_mapping.hql
   # or, using Beeline with Kerberos:
   beeline -u "jdbc:hive2://<hs2-host>:10000/mnaas;principal=hive/_HOST@REALM" -f bl_cust_bu_co_cogrp_mapping.hql
   ```

2. **Verification steps**  
   ```sql
   -- Verify table exists
   SHOW CREATE TABLE mnaas.bl_cust_bu_co_cogrp_mapping;

   -- Sample data check
   SELECT COUNT(*) AS rows, MIN(start_bill_date) AS first_date FROM mnaas.bl_cust_bu_co_cogrp_mapping;
   ```

3. **Debugging tips**  
   - **Metastore errors**: Check HiveServer2 logs (`/var/log/hive/hiveserver2.log`).  
   - **HDFS path problems**: Run `hdfs dfs -ls <path>` to confirm directory visibility.  
   - **SerDe parsing issues**: Run a `SELECT * FROM … LIMIT 5` and inspect column values; mismatched delimiters will appear as concatenated fields.  
   - **Impala visibility**: In Impala shell, execute `SHOW TABLES LIKE 'bl_cust_bu_co_cogrp_mapping';` and `DESCRIBE FORMATTED …;`.

---

### 7. External Configuration & Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `HIVE_HOME` | Determines Hive client binaries used to run the script. |
| `HADOOP_CONF_DIR` | Provides HDFS namenode address (`NN-HA1`) and security settings. |
| `HIVE_METASTORE_URI` | Points to the metastore service where the table definition is stored. |
| `IMPALA_CATALOG_SERVICE_ID` (referenced in table properties) | Used by Impala to map the table to its catalog version; not required at DDL time but important for downstream Impala queries. |
| `HDFS_USER` (often `hdfs` or `root`) | The user that must have read/write permissions on the external location. |
| **No hard‑coded values** in the script besides the HDFS URI; any change to the cluster name or warehouse path should be handled via environment variables or a wrapper script that substitutes the location before execution. |

---

### 8. Suggested Improvements (TODO)

1. **Add explicit column comments and proper data types** – Replace generic `string` for date fields (`co_mapping_start_date`, `co_mapping_end_date`, `start_bill_date`) with `date` or `timestamp` to enable native date functions and validation.  
2. **Introduce partitioning** – If the data volume grows, partition the table by `co_mapping_start_date` (or `org_id`) to improve query performance and enable incremental loads.  

--- 

*End of documentation.*