**File:** `mediation-ddls\mnaas_suspended_sim_list.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates (or replaces) an **external, partition‑by‑month** table named `mnaas.suspended_sim_list`. The table stores records of SIM cards that have been suspended, including identifiers, commercial offer, suspension/reactivation dates, and a month partition key. Because the table is external and points to a fixed HDFS location, the data lifecycle is managed outside Hive (e.g., by upstream ETL jobs that drop files into the directory). The table is intended for downstream reporting, analytics, and validation steps within the mediation layer of the telecom data‑move pipeline.

---

## 2. Important Objects & Their Responsibilities

| Object | Type | Responsibility |
|--------|------|-----------------|
| `suspended_sim_list` | Hive external table | Holds raw suspension records; partitioned by `month` for efficient pruning. |
| Columns (`sim`, `msisdn`, `tcl_secs_id`, `orgname`, `commercialoffer`, `suspended_date`, `activated`, `reactivateddate`) | Table schema | Capture the key attributes needed by downstream reports (e.g., usage, revenue, churn). |
| `month` | Partition column | Enables time‑based slicing; each month is a separate HDFS sub‑directory. |
| `LOCATION 'hdfs://NN-HA1/.../suspended_sim_list'` | HDFS path | Physical storage location for the external data files. |
| Table properties (`DO_NOT_UPDATE_STATS`, `external.table.purge`, etc.) | Hive/Impala metadata | Control statistics collection, automatic purge on DROP, and Impala catalog integration. |

*No procedural code (functions, procedures) is present; the script is purely declarative.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - The HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/suspended_sim_list` must exist (or be creatable). <br> - Data files placed there must be plain‑text, delimited in a way compatible with `LazySimpleSerDe` (default is `\001` field delimiter). |
| **Outputs** | - A Hive metastore entry for `mnaas.suspended_sim_list`. <br> - Impala catalog entries (via the `impala.events.*` properties). |
| **Side‑Effects** | - If the table already exists, Hive will replace the definition (metadata only; underlying files remain untouched). <br> - Because `external.table.purge='true'`, dropping the table will also delete the underlying HDFS files. |
| **Assumptions** | - The downstream ETL that populates the directory follows the same column order and data types. <br> - The `month` partition directories are created as `month=YYYYMM` (or similar) matching the partition value in the data. <br> - No statistics are required for this raw table (`DO_NOT_UPDATE_STATS='true'`). |
| **External Services** | - HDFS NameNode (`NN-HA1`). <br> - Hive Metastore service. <br> - Impala catalog service (referenced by `impala.events.catalogServiceId`). |

---

## 4. Integration with Other Scripts & Components

| Connected Component | How It Links |
|---------------------|--------------|
| **Downstream reporting DDLs** (e.g., `mnaas_ra_*` tables) | Queries join `suspended_sim_list` to filter out suspended SIMs or to enrich usage reports. |
| **Ingestion jobs** (not shown) | Likely a separate Spark/MapReduce or Sqoop job that writes CSV/TSV files into the HDFS location, possibly partitioned by month. |
| **Data quality / validation scripts** | May read the table to verify that every suspended SIM appears in the source system. |
| **Impala UI / BI tools** | Users query the table directly via Impala for ad‑hoc analysis. |
| **Orchestration layer (e.g., Airflow, Oozie)** | A DAG/task runs this DDL before any job that expects the table to exist, ensuring the metastore is up‑to‑date. |

*Because the table is external, any process that adds or removes files must respect the partition layout (`/month=.../`).*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale or missing partitions** | Queries may scan empty or wrong data, leading to inaccurate reports. | Implement a partition‑refresh job (`MSCK REPAIR TABLE` or `ALTER TABLE ... ADD PARTITION`) after each data load. |
| **Accidental data loss on DROP** | `external.table.purge='true'` will delete underlying files if the table is dropped. | Restrict DROP privileges; use a “soft‑drop” (rename) process instead of DROP in production. |
| **Incorrect delimiter / schema drift** | Data parsing errors, null values, or truncated rows. | Enforce a schema validation step in the ingestion pipeline; consider adding `SERDEPROPERTIES ('field.delim'=',')` explicitly. |
| **Impala catalog out‑of‑sync** | Queries via Impala may see stale metadata. | Run `INVALIDATE METADATA` or `REFRESH` after DDL execution; monitor Impala catalog events. |
| **Permissions mismatch on HDFS path** | Jobs may fail to write/read files. | Ensure the Hive/Impala service accounts have `rwx` on the directory; audit ACLs regularly. |

---

## 6. Running & Debugging the Script

1. **Execution (standard)**  
   ```bash
   hive -f mediation-ddls/mnaas_suspended_sim_list.hql
   # or, for Impala:
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_suspended_sim_list.hql
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.suspended_sim_list;
   DESCRIBE FORMATTED mnaas.suspended_sim_list;
   SELECT COUNT(*) FROM mnaas.suspended_sim_list LIMIT 10;
   ```

3. **Partition Check**  
   ```sql
   SHOW PARTITIONS mnaas.suspended_sim_list;
   -- If missing partitions:
   MSCK REPAIR TABLE mnaas.suspended_sim_list;
   ```

4. **Debugging Tips**  
   * If Hive reports “Table already exists”, use `DROP TABLE IF EXISTS mnaas.suspended_sim_list;` before re‑creating (only metadata, not data).  
   * If queries return no rows, verify that HDFS contains files under `.../suspended_sim_list/month=YYYYMM/`.  
   * Check the Hive metastore logs for serialization errors (often caused by unexpected delimiters).  
   * For Impala, run `INVALIDATE METADATA mnaas.suspended_sim_list;` after the DDL.

---

## 7. External Configuration / Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `NN-HA1` (NameNode hostname) | Hard‑coded in the `LOCATION` clause; may be overridden by a cluster‑wide HDFS alias or DNS. |
| Hive/Impala connection parameters (e.g., `hive.metastore.uris`, `impala.catalog.service.id`) | Not referenced directly in the script but required for the client that runs the DDL. |
| Optional: `HIVE_CONF_DIR`, `IMPALA_HOST` | Environment variables used by the execution wrapper (Airflow, Oozie) to locate configuration files. |

If the deployment uses a templating system (e.g., Jinja, Velocity), the `LOCATION` string could be parameterised; verify the CI/CD pipeline for such substitutions.

---

## 8. Suggested TODO / Improvements

1. **Explicit delimiter & null handling** – Add `ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe' WITH SERDEPROPERTIES ('field.delim'=',', 'serialization.null.format'='')` to make the file format deterministic and avoid reliance on Hive defaults.

2. **Automated partition management** – Create a companion script (e.g., `add_month_partition.sh`) that runs after each data load:
   ```bash
   hive -e "ALTER TABLE mnaas.suspended_sim_list ADD IF NOT EXISTS PARTITION (month='${YYYYMM}') LOCATION 'hdfs://NN-HA1/.../suspended_sim_list/month=${YYYYMM}';"
   ```
   This prevents missing‑partition errors and removes the need for `MSCK REPAIR TABLE` in production. 

---