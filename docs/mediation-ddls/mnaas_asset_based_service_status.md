**File:** `mediation-ddls\mnaas_asset_based_service_status.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the **`asset_based_service_status`** table in the `mnaas` database. The table stores raw asset‑based service events (e.g., SIM activation, service status changes) with a string‑based schema and is partitioned by `event_month`. It is defined as an *insert‑only transactional* table stored under a managed Hive warehouse location on HDFS (`hdfs://NN-HA1/...`). The table is a foundational data store for downstream mediation, reporting, and analytics pipelines that consume asset‑based service status records.

---

## 2. Key Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `createtab_stmt` | Variable holding the DDL statement | Encapsulates the full `CREATE TABLE` definition; used by the execution engine (e.g., Hive CLI, Oozie, Airflow) to materialize the table. |
| `mnaas.asset_based_service_status` | Hive table | Persists raw event records with columns such as `sourceid`, `eventid`, `eventtimestamp`, `entityid_iccid`, `imsi`, etc. |
| `event_month` | Partition column (string) | Enables time‑based pruning; each month of data lands in a separate HDFS sub‑directory. |
| Table properties (`transactional='true'`, `transactional_properties='insert_only'`) | Hive metadata | Guarantees ACID‑compatible inserts while disallowing updates/deletes, suitable for immutable event logs. |
| `LOCATION` | HDFS path | Physical storage location for the table data; points to the managed Hive warehouse on the primary NameNode (`NN-HA1`). |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | No external data files are read by this script. It relies on Hive metastore configuration and the HDFS cluster reachable via `NN-HA1`. |
| **Outputs** | - A new Hive table `mnaas.asset_based_service_status` (or replacement if it already exists). <br> - Corresponding HDFS directory structure (`.../asset_based_service_status/event_month=...`). |
| **Side Effects** | - Updates Hive metastore metadata. <br> - May create the HDFS directory hierarchy if it does not exist. |
| **Assumptions** | - Hive version supports transactional *insert‑only* tables (Hive 2.3+). <br> - The Hive warehouse base (`hdfs://NN-HA1/warehouse/tablespace/managed/hive/`) is writable by the Hive service user. <br> - `NN-HA1` resolves to the active NameNode (often defined in a cluster‑wide config file). <br> - No existing table with the same name that would cause an unwanted overwrite (the script does **not** use `IF NOT EXISTS`). |
| **External Services** | - Hadoop HDFS (NameNode `NN-HA1`). <br> - Hive Metastore service. |

---

## 4. Integration Points  

| Connected Component | How the Table Is Used |
|---------------------|-----------------------|
| **Ingestion scripts** (e.g., `mnaas_asset_based_service_status_load.hql` or Spark jobs) | Load raw event files into this table, typically partitioned by `event_month`. |
| **Transformation pipelines** (e.g., `mnaas_asset_based_service_status_enrich.hql`) | Read from this table to enrich with reference data (e.g., device inventory) and write to downstream fact tables. |
| **Reporting / BI** (e.g., Tableau, PowerBI connectors) | Query the table directly for ad‑hoc analysis of service status trends. |
| **Data quality / monitoring jobs** (e.g., Oozie workflow `asset_status_qc.xml`) | Validate row counts, partition completeness, and schema conformity. |
| **Retention / archival jobs** | Drop old partitions (`ALTER TABLE … DROP PARTITION`) based on business retention policy. |

*Note:* The exact script names are inferred from the naming convention used in the surrounding `mediation-ddls` directory (e.g., `mnaas_*`). Verify actual job definitions in the orchestration layer (Oozie, Airflow, etc.) for precise connections.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Accidental table overwrite** (script re‑run without `IF NOT EXISTS`) | Loss of metadata; downstream jobs may fail or read stale data. | Add `IF NOT EXISTS` to the `CREATE TABLE` statement or guard the execution with a pre‑check (`SHOW TABLES LIKE 'asset_based_service_status'`). |
| **Unbounded partition growth** (string `event_month` may contain malformed values) | HDFS namespace explosion, query performance degradation. | Enforce a strict `YYYYMM` format via downstream ingestion validation; consider using `DATE` type partition column. |
| **Insufficient HDFS storage** (large raw event volume) | Job failures, data loss. | Monitor HDFS usage; set alerts on the `asset_based_service_status` directory; implement retention policies. |
| **Transactional insert‑only limitation** (cannot update/delete rows) | If a correction is needed, you must rewrite the partition. | Document the limitation; provide a “reprocess partition” utility for error correction. |
| **NameNode address hard‑coded (`NN-HA1`)** | Breaks when cluster topology changes. | Externalize the NameNode URI to a configuration file or environment variable (e.g., `HDFS_NN_URI`). |

---

## 6. Running / Debugging the Script  

1. **Standard execution** (via Hive CLI):  
   ```bash
   hive -f mediation-ddls/mnaas_asset_based_service_status.hql
   ```
2. **Within an Oozie workflow** – reference the file as an `<action>` of type `hive`. Ensure the workflow’s `<job-tracker>` and `<name-node>` match the cluster configuration.  

3. **Debug steps**  
   - **Check table existence**: `SHOW CREATE TABLE mnaas.asset_based_service_status;`  
   - **Validate HDFS location**: `hdfs dfs -ls hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/asset_based_service_status`  
   - **Inspect partitions**: `SHOW PARTITIONS mnaas.asset_based_service_status;`  
   - **Review Hive logs** (`/var/log/hive/hive.log`) for any DDL errors or permission issues.  

4. **Re‑run safely** (idempotent):  
   ```sql
   CREATE TABLE IF NOT EXISTS mnaas.asset_based_service_status ( ... ) PARTITIONED BY (event_month STRING) ... ;
   ```

---

## 7. External Configuration & Environment Variables  

| Config / Variable | Usage |
|-------------------|-------|
| `NN-HA1` | Hostname of the active HDFS NameNode; appears in the `LOCATION` clause. Typically defined in cluster‑wide Hadoop configuration (`core-site.xml`). |
| Hive warehouse directory (`hive.metastore.warehouse.dir`) | Determines the base path (`/warehouse/tablespace/managed/hive/`). The script assumes the default managed location. |
| Hive execution engine settings (e.g., `hive.exec.dynamic.partition.mode`) | Must allow dynamic partition creation if downstream loads write to new `event_month` partitions. |
| Environment variables for orchestration (e.g., `HIVE_CONF_DIR`) | Needed when the script is launched from a scheduler (Oozie, Airflow). |

If any of these values differ across environments (dev, test, prod), they are usually overridden via a properties file or environment‑specific wrapper script.

---

## 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend `IF NOT EXISTS` and optionally add a `DROP TABLE IF EXISTS` guard with a versioned backup strategy.  
2. **Refine data types** – change columns like `eventtimestamp`, `start_time`, `created_on`, `last_updated_on` from `STRING` to `TIMESTAMP` (or `BIGINT` epoch) to enable proper time‑based queries and partition pruning.  

Implementing these changes will improve data quality, query performance, and operational safety.