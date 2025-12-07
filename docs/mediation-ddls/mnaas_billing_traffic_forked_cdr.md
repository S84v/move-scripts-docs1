**File:** `mediation-ddls\mnaas_billing_traffic_forked_cdr.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the external table **`mnaas.billing_traffic_forked_cdr`**. The table stores raw Call Detail Record (CDR) data that has been “forked” (i.e., duplicated or split) for downstream billing calculations. Data resides on HDFS under the `mnaas.db/billing_traffic_forked_cdr` directory and is partitioned by `bill_month` (string) and `tcl_secs_id` (int). The table is read‑only from a production perspective – downstream ETL jobs query it, while upstream mediation processes write raw files into the HDFS location.

---

## 2. Core Object(s) Defined
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_traffic_forked_cdr` | Hive **EXTERNAL** table | Provides a schema‑aware view over raw forked‑CDR files stored in HDFS; enables partition pruning for billing‑related queries. |
| `createtab_stmt` (script variable) | DDL string | Holds the full `CREATE EXTERNAL TABLE` statement; used by the deployment framework to execute the DDL. |

*No procedural code (functions, classes) is present in this file; it is purely declarative.*

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Input data** | Raw text files (delimited by default Hive SerDe) placed in the HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/billing_traffic_forked_cdr`. Files must conform to the column order defined in the DDL. |
| **Output** | Hive metastore entry for the external table; queryable virtual table `mnaas.billing_traffic_forked_cdr`. No data is copied or transformed by this script. |
| **Side effects** | - Registers table metadata (columns, partitions, location). <br>- Enables automatic partition discovery when new `bill_month`/`tcl_secs_id` directories appear (if `MSCK REPAIR TABLE` is later run). |
| **Assumptions** | - HDFS path exists and is writable by the Hive service account. <br>- Files are UTF‑8 text, fields delimited by the default Hive delimiter (Ctrl‑A). <br>- Partition columns are present as directory names (`bill_month=YYYYMM`, `tcl_secs_id=N`). <br>- Impala catalog is kept in sync via the listed `impala.events.*` properties. |

---

## 4. Integration Points  

| Connected Component | Interaction |
|---------------------|-------------|
| **Upstream mediation jobs** (e.g., `mnaas_billing_traffic_filtered_cdr.hql` pipeline) | Write raw forked‑CDR files into the HDFS location; may also invoke `MSCK REPAIR TABLE` to refresh partitions. |
| **Downstream billing ETL** (e.g., `mnaas_billing_traffic_actusage.hql`, `mnaas_billing_product_activation.hql`) | Query this table to calculate usage, apply tariffs, and generate billing events. |
| **Impala** | Reads the same external table for low‑latency analytics; the `impala.events.*` properties ensure catalog updates propagate. |
| **Orchestration layer** (Airflow / Oozie) | Executes this DDL as part of a “DDL bootstrap” task before any data load jobs run. |
| **Monitoring / Auditing** | Hive metastore logs capture the DDL execution; external tools may query `tblproperties` for audit trails (`last_modified_by`, `last_modified_time`). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream processes write files with missing/extra columns. | Query failures, incorrect billing. | Enforce schema validation in the ingestion job (e.g., using Apache NiFi or a Spark schema‑check step). |
| **Stale partitions** – new `bill_month` directories not discovered. | Data for a new month is invisible to queries. | Schedule a periodic `MSCK REPAIR TABLE mnaas.billing_traffic_forked_cdr;` or enable `hive.msck.repair.partition` auto‑repair. |
| **Uncontrolled data growth** – external table purge is true, but orphan files may accumulate. | Disk pressure on HDFS. | Implement a retention policy (e.g., delete partitions older than N months) via a scheduled cleanup script. |
| **Permission mismatch** – Hive service account loses write access to the HDFS location. | Table becomes unreadable. | Include ACL checks in the deployment pipeline; document required HDFS permissions (`rwx` for hive:hive). |
| **Impala catalog desynchronization** – catalog version lag. | Queries return stale metadata. | Verify that `impala-shell -i <impala-host> -q "invalidate metadata mnaas.billing_traffic_forked_cdr;"` runs after DDL changes. |

---

## 6. Running / Debugging the Script  

1. **Execution** (typical operator command)  
   ```bash
   hive -f mediation-ddls/mnaas_billing_traffic_forked_cdr.hql
   # or via Beeline
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaas_billing_traffic_forked_cdr.hql
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.billing_traffic_forked_cdr;
   DESCRIBE FORMATTED mnaas.billing_traffic_forked_cdr;
   SELECT COUNT(*) FROM mnaas.billing_traffic_forked_cdr LIMIT 1;
   ```

3. **Debugging Tips**  
   - Check Hive metastore logs (`/var/log/hive/hive-metastore.log`) for DDL errors.  
   - Verify the HDFS directory exists: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/billing_traffic_forked_cdr`.  
   - If partitions are missing, run `MSCK REPAIR TABLE mnaas.billing_traffic_forked_cdr;` and re‑query.  
   - For Impala visibility issues, run `invalidate metadata` from `impala-shell`.  

---

## 7. External Configuration & Environment Variables  

| Config / Env | Purpose |
|--------------|---------|
| `HIVE_CONF_DIR` / `HIVE_HOME` | Hive client configuration (metastore URI, Kerberos tickets). |
| `NN-HA1` (namenode HA address) | Resolved by the Hadoop client; must be reachable from the Hive server. |
| `HADOOP_USER_NAME` (often `hive`) | Determines HDFS access rights for the external table location. |
| `TABLE_LOCATION` (hard‑coded in DDL) | If the environment uses a different warehouse root, the DDL must be edited or templated. |
| `impala.events.catalogServiceId` & `catalogVersion` | Managed by Impala; not manually set but may be overridden in a test environment. |

No additional property files are referenced directly by this script.

---

## 8. Suggested Improvements (TODO)

1. **Add column comments & data‑type tightening** – e.g., store `calltimestamp` as `BIGINT` epoch if source format is numeric; document each field for downstream developers.  
2. **Automate partition discovery** – embed a `MSCK REPAIR TABLE` call in the deployment pipeline or enable `hive.msck.repair.partition` = `true` in Hive configuration to avoid manual refresh.  

---