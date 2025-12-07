**File:** `mediation-ddls\mnaas_active_eid_list.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive external table `mnaas.active_eid_list`. The table stores a flat‑file list of active **EID** (Enterprise ID) records, partitioned by month, and is persisted in HDFS under the `mnaas.db/active_eid_list` directory. Down‑stream mediation jobs read this table to enrich or filter records for billing, activation, and reporting pipelines.

---

## 2. Core Artifact

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `CREATE EXTERNAL TABLE mnaas.active_eid_list` | Hive DDL | Defines schema (`eid`, `tcl_secs_id`, `orgname`, `commercialoffer`), monthly partitioning, storage format (LazySimpleSerDe / TextInputFormat), and HDFS location. |
| Table properties | Hive metadata | Enables automatic purge of files on DROP, tracks Impala catalog IDs, and records audit timestamps. |

*No procedural code (functions, classes) exists in this file; it is a pure DDL definition.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | - Implicit: Hive/Impala metastore connection.<br>- Implicit: HDFS namenode address (e.g., `hdfs://NN-HA1`). |
| **Outputs** | - Hive external table metadata registered in the `mnaas` database.<br>- Physical directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/active_eid_list` (created if missing). |
| **Side Effects** | - If the table already exists, Hive will error unless `IF NOT EXISTS` is added (currently not present).<br>- Because `external.table.purge='true'`, dropping the table will also delete underlying files. |
| **Assumptions** | - HDFS path is reachable and has write permission for the Hive service user (often `hive` or `root`).<br>- The `mnaas` database already exists.<br>- Down‑stream jobs will load data into the `month` partitions before querying. |

---

## 4. Integration Points

| Component | Interaction |
|-----------|-------------|
| **Ingestion jobs** (e.g., `mnaas_active_eid_load.hql` or Spark/MapReduce loaders) | Write raw CSV/TSV files into the HDFS location, then `ALTER TABLE … ADD PARTITION (month='YYYYMM') LOCATION …` to make them visible. |
| **Transformation pipelines** (e.g., `mnaas_activations_raw_daily.hql`) | Join on `active_eid_list` to filter only active EIDs. |
| **Reporting / BI** (Impala, Tableau, etc.) | Query the table directly; Impala catalog IDs stored in table properties keep the metadata in sync. |
| **Data‑quality monitors** | Expect the `month` partition to be present for each processing cycle; missing partitions trigger alerts. |
| **Cleanup scripts** | May drop/recreate the table during environment refreshes; must respect the purge flag. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental table drop** → loss of raw files (purge flag). | Data loss, downstream job failures. | Restrict DROP privileges; add a pre‑drop approval step; consider removing `external.table.purge='true'` in non‑prod. |
| **Partition drift** – partitions not added after data load. | Queries return empty result sets. | Implement a post‑load validation job that checks for expected `month` partitions. |
| **HDFS permission mismatch** – Hive user cannot write to the location. | Table creation fails or later loads fail. | Verify directory ACLs (`hdfs dfs -ls -R …`) and ensure Hive service user has `rw` rights. |
| **Schema mismatch** – upstream loaders produce columns in a different order or type. | Query errors or silent data corruption. | Enforce schema validation in the loader (e.g., using Spark schema enforcement) before writing files. |
| **Stale Impala catalog** – Impala may not see new partitions immediately. | Delayed data availability for BI. | Run `INVALIDATE METADATA mnaas.active_eid_list` or `REFRESH` after partition addition. |

---

## 6. Execution & Debugging Guide

1. **Run the script**  
   ```bash
   hive -f mediation-ddls/mnaas_active_eid_list.hql
   # or via Beeline
   beeline -u jdbc:hive2://<hs2-host>:10000/default -f mediation-ddls/mnaas_active_eid_list.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.active_eid_list;
   DESCRIBE FORMATTED mnaas.active_eid_list;
   ```

3. **Check HDFS location**  
   ```bash
   hdfs dfs -ls -R /user/hive/warehouse/mnaas.db/active_eid_list
   ```

4. **Debug common failures**  
   - *Error: Table already exists* → add `IF NOT EXISTS` or drop first.  
   - *Permission denied* → inspect HDFS ACLs, adjust `hdfs dfs -chmod`/`-chown`.  
   - *Metastore connectivity* → verify HiveServer2 URL and Kerberos tickets (if enabled).

5. **Log locations**  
   - HiveServer2 logs (`/var/log/hive/hiveserver2.log`).  
   - Impala catalog logs for table‑property changes.

---

## 7. External Configuration & Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Locate `hive-site.xml` and `core-site.xml` for metastore & HDFS connection. |
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` clause; may be overridden by a cluster‑wide DNS alias. |
| `HIVE_DATABASE` (if scripted) | Not used directly here, but downstream scripts may reference the `mnaas` DB name. |
| Impala catalog IDs (`impala.events.catalogServiceId`, `catalogVersion`) | Populated automatically; no manual config needed. |

If the environment changes (e.g., a new namenode address), the `LOCATION` string must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Make table creation idempotent** – prepend `IF NOT EXISTS` to avoid failures on re‑run, and optionally `DROP TABLE IF EXISTS` with a safety guard for non‑prod environments.  
2. **Externalize the HDFS base path** – replace the hard‑coded `hdfs://NN-HA1/...` with a variable (e.g., `${hdfs.base}`) read from a central config file, enabling smoother migrations between clusters.  

---