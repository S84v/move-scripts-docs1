**File:** `mediation-ddls\mnaas_addon_subs_aggr.hql`  

---

## 1. High‑Level Summary
This HQL script creates an **external Hive/Impala table** named `mnaas.addon_subs_aggr`. The table stores aggregated add‑on subscription records (e.g., ICCID, business unit, dates, addon details, amount, etc.) in a plain‑text location on HDFS. Because the table is external and marked with `external.table.purge='true'`, Hive will automatically delete the underlying files when the table is dropped, making it a lightweight staging/aggregation surface for downstream mediation jobs.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.addon_subs_aggr` | External Hive table | Holds raw/aggregated add‑on subscription data for the **MNAAS** mediation domain. |
| Columns (`tcl_secs_id`, `addon_name`, … `unit_type`) | Table schema | Capture identifiers, business‑unit metadata, temporal boundaries, and financial metrics for each add‑on record. |
| `ROW FORMAT SERDE` | SerDe definition | Uses `LazySimpleSerDe` to parse delimited text (default delimiter = `\001`). |
| `INPUTFORMAT / OUTPUTFORMAT` | File format | Text input, Hive‑ignore‑key output – compatible with Impala. |
| `LOCATION` | HDFS path | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/addon_subs_aggr` – the physical storage location. |
| `TBLPROPERTIES` | Metadata | Includes purge flag, Impala catalog IDs, timestamps, and audit fields (`last_modified_by`, `last_modified_time`). |

*No procedural code (functions, procedures) is present; the file is a pure DDL statement.*

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Inputs** | Implicit – the script expects the HDFS directory `.../addon_subs_aggr` to be reachable and writable by the Hive/Impala service user (typically `hive` or `impala`). |
| **Outputs** | Creation of the external table metadata in the Hive metastore. No data is loaded by this script; downstream jobs will write files into the defined location. |
| **Side‑effects** | - Registers a new table in the metastore.<br>- Because `external.table.purge='true'`, dropping the table will delete the underlying files.<br>- Updates Impala catalog events (catalogServiceId, catalogVersion). |
| **Assumptions** | - Hive/Impala services are running on a cluster where `NN-HA1` resolves to the active NameNode.<br>- The Hive metastore is reachable and the user executing the script has `CREATE` privileges on database `mnaas`.<br>- The target HDFS directory either does not exist (Hive will create it) or is empty. |

---

## 4. Integration with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Downstream ETL jobs** (e.g., `mnaas_addon_subs_load.hql`, `mnaas_addon_subs_transform.py`) | These jobs write aggregated CSV/TSV files into the table’s HDFS location and later query `mnaas.addon_subs_aggr` for reporting or further enrichment. |
| **Reporting / BI layer** (Impala/Presto queries) | Analysts query this table directly for add‑on revenue dashboards. |
| **Data retention / purge processes** (e.g., `mnaas_cleanup.hql`) | May drop and recreate the table to enforce retention; the purge flag ensures physical data removal. |
| **Orchestration engine** (Oozie, Airflow, or custom scheduler) | The DDL is typically executed as a “create‑table” task in a workflow that runs before any load step. |
| **Metadata catalog** (Cloudera Navigator / Atlas) | Table properties (`impala.events.*`) allow the catalog to track versioning and lineage. |

*Because the file lives in `mediation-ddls`, it is part of the “DDL bundle” that other mediation scripts reference via fully‑qualified table names (`mnaas.addon_subs_aggr`).*

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss** – `external.table.purge='true'` causes physical files to be deleted when the table is dropped. | Loss of raw aggregation files. | Enforce a change‑control policy: require explicit approval before dropping the table; optionally disable purge in non‑prod environments. |
| **Schema drift** – downstream jobs may write columns that do not match the defined schema (e.g., new fields). | Query failures, silent data truncation. | Add a schema‑validation step in the load job; version the table (e.g., `addon_subs_aggr_v2`). |
| **HDFS path permission issues** – the Hive/Impala user may lack write access. | Table creation fails; downstream jobs cannot write. | Verify HDFS ACLs before deployment; include a pre‑flight script that checks `hdfs dfs -test -d` and `-w`. |
| **NameNode address change** – hard‑coded `NN-HA1` may become stale after HA re‑configuration. | Table creation points to wrong namenode, causing I/O errors. | Externalize the namenode URI into a config property (e.g., `${HDFS_NN_URI}`) and reference it via Hive variables. |
| **Missing/incorrect SerDe settings** – default delimiter may not match source files. | Data parsing errors. | Document the expected delimiter; consider adding `FIELDS TERMINATED BY '\t'` if needed. |

---

## 6. Running / Debugging the Script  

1. **Typical execution** (via Hive CLI or Beeline):  
   ```bash
   hive -f mediation-ddls/mnaas_addon_subs_aggr.hql
   # or
   beeline -u jdbc:hive2://<host>:10000 -f mediation-ddls/mnaas_addon_subs_aggr.hql
   ```

2. **Airflow/Oozie task** – define a `HiveOperator`/`ShellAction` that points to the file path.

3. **Verification steps**:  
   - After execution, run `SHOW CREATE TABLE mnaas.addon_subs_aggr;` to confirm the DDL.  
   - Check HDFS location: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/addon_subs_aggr`.  
   - Query a sample: `SELECT COUNT(*) FROM mnaas.addon_subs_aggr LIMIT 10;` (should return 0 rows until data is loaded).  

4. **Debugging**:  
   - If the script fails with “Table already exists”, add `IF NOT EXISTS` to the `CREATE EXTERNAL TABLE` clause.  
   - For permission errors, inspect the Hive metastore logs and HDFS audit logs.  
   - Use `hive --hiveconf hive.exec.dynamic.partition=true` if later jobs rely on dynamic partitions (not currently defined).  

---

## 7. External Configuration / Environment Variables  

| Variable / File | Usage |
|-----------------|-------|
| `NN-HA1` (hard‑coded in the `LOCATION` URI) | NameNode hostname for the HDFS cluster. Ideally should be injected via a Hive variable (`${hdfs.nn}`) rather than hard‑coded. |
| Hive/Impala metastore connection settings (e.g., `hive.metastore.uris`) | Required for the DDL to be persisted. Not referenced directly in the script but must be configured in the execution environment. |
| Optional: `HIVE_CONF_DIR`, `HADOOP_CONF_DIR` | Provide Hadoop core and Hive configuration files for the client executing the script. |

---

## 8. Suggested TODO / Improvements  

1. **Add `IF NOT EXISTS`** to make the script idempotent:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.addon_subs_aggr ( ... )
   ```

2. **Externalize the HDFS URI** and consider partitioning by `start_month`/`end_month` to improve query performance and enable incremental loads:  
   ```sql
   PARTITIONED BY (start_month STRING, end_month STRING)
   LOCATION '${hdfs.base}/mnaas/addon_subs_aggr'
   ```

3. **Replace generic `string` types** for numeric fields (`amount`) with appropriate numeric types (`DECIMAL(15,2)`) to enable proper aggregation and avoid casting at query time.

---