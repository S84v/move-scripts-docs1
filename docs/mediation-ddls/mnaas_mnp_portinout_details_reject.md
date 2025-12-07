**File:** `mediation-ddls\mnaas_mnp_portinout_details_reject.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the **`mnaas.mnp_portinout_details_reject`** table, which stores records of Mobile Number Portability (MNP) “port‑in/out” transactions that were rejected during mediation. Each row captures the original transaction payload (IDs, subscriber data, porting dates, operator codes, source file name, insertion timestamp) together with a textual reject reason. The table is defined as an *insert‑only transactional* Hive table stored as plain‑text files on HDFS under the managed warehouse path.

---

## 2. Core Object(s) Defined
| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.mnp_portinout_details_reject` | Hive Managed Table | Persists rejected MNP port‑in/out detail records for downstream reporting, audit, and re‑processing. |
| Columns (15) | Schema fields | Capture the full payload of the original transaction plus a `reject_reason` and Hive‑managed metadata (`record_insertime`). |
| Table Properties | Hive metadata | `transactional='true'` & `insert_only` enable ACID‑style inserts without updates/deletes; `bucketing_version='2'` prepares for future bucketing if needed. |
| SerDe & Format | `LazySimpleSerDe` with `field.delim=';'` | Guarantees compatibility with upstream flat‑file producers that emit semicolon‑delimited records. |
| Location | HDFS path `hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/mnp_portinout_details_reject` | Physical storage location for the table’s data files. |

---

## 3. Inputs, Outputs & Side Effects
| Aspect | Details |
|--------|---------|
| **Inputs** | – Implicit: upstream mediation jobs write rejected records (semicolon‑delimited) to the HDFS location defined above. <br>– No explicit parameters; the script only creates the table. |
| **Outputs** | – Hive metastore entry for `mnp_portinout_details_reject`. <br>– Physical directory on HDFS where subsequent INSERT statements will land data files. |
| **Side Effects** | – If the table already exists, Hive will **replace** it (default behavior without `IF NOT EXISTS`). <br>– Creates the target HDFS directory if it does not exist. |
| **Assumptions** | – HiveServer2 is reachable and the executing user has `CREATE` privileges on the `mnaas` database. <br>– HDFS namenode alias `NN-HA1` resolves correctly in the cluster. <br>– Upstream jobs respect the `;` field delimiter and newline record delimiter. |

---

## 4. Integration Points with Other Scripts / Components
| Connected Component | Relationship |
|---------------------|--------------|
| `mnaas_mnp_portinout_details_raw.hql` | Populates the *raw* MNP port‑in/out table; downstream validation may move rejected rows into this reject table. |
| `mnaas_mnp_portinout_details_reject.hql` (this file) | Consumed by orchestration (e.g., Oozie, Airflow, or custom scheduler) that runs the DDL before any INSERT jobs. |
| ETL jobs that perform **reject detection** (e.g., validation scripts, Spark/MapReduce jobs) | Insert rows into `mnp_portinout_details_reject` using `INSERT INTO TABLE … SELECT … WHERE reject_flag = true`. |
| Reporting / audit tools | Query this table to generate rejection statistics, SLA dashboards, or to trigger re‑processing. |
| Data Lake / downstream data warehouse | May ingest this table via Hive‑to‑Impala, Spark, or export to Parquet for long‑term storage. |

---

## 5. Operational Risks & Recommended Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Table recreation overwrites existing data** (script run without `IF NOT EXISTS`) | Loss of historic reject records. | Add `IF NOT EXISTS` or guard the DDL with a pre‑check (`SHOW TABLES LIKE ...`). |
| **Schema drift** – upstream producers change delimiter or add columns | Insert jobs fail, data becomes unreadable. | Enforce schema versioning; add a validation step that reads a sample file before insertion. |
| **HDFS path mis‑configuration** (wrong namenode alias or missing directory) | Table creation fails; downstream jobs stall. | Centralise HDFS base path in a config file (e.g., `hive-site.xml` or environment variable) and reference it via `${hdfs.base}`. |
| **Transactional table performance** – insert‑only tables can cause small file proliferation. | Degraded query performance, increased NameNode load. | Periodic compaction (Hive `ALTER TABLE … COMPACT 'MAJOR'`) scheduled via workflow. |
| **Insufficient permissions** (user cannot create transactional tables) | Script aborts, pipeline stops. | Document required Hive roles; include a pre‑flight permission check in the orchestration. |

---

## 6. Running / Debugging the Script
1. **Typical execution (via orchestration)**  
   ```bash
   hive -f mediation-ddls/mnaas_mnp_portinout_details_reject.hql
   ```
   The orchestration framework (e.g., Airflow `HiveOperator`) will capture stdout/stderr and mark the task as success/failure.

2. **Manual run (developer)**  
   - Open a Hive CLI or Beeline session with the appropriate Kerberos ticket.  
   - Execute the script file or paste the DDL.  
   - Verify creation: `SHOW CREATE TABLE mnaas.mnp_portinout_details_reject;`

3. **Debugging steps**  
   - **Check Hive logs** (`/var/log/hive/hive-server2.log`) for syntax errors or permission denials.  
   - **Validate HDFS location**: `hdfs dfs -ls /warehouse/tablespace/managed/hive/mnaas.db/mnp_portinout_details_reject`.  
   - **Confirm table properties**: `DESCRIBE FORMATTED mnaas.mnp_portinout_details_reject;` – ensure `transactional` and `insert_only` are set.  
   - **Test insert**: run a small `INSERT INTO TABLE … VALUES (…)` to confirm the SerDe parses the semicolon delimiter correctly.

---

## 7. External Configuration / Environment Variables
| Config Item | Usage |
|------------|-------|
| `NN-HA1` (HDFS namenode alias) | Hard‑coded in the `LOCATION` clause; typically defined in the cluster’s `core-site.xml`. |
| Hive warehouse root (`hive.metastore.warehouse.dir`) | Implicitly used to resolve the managed table location; should point to the same HDFS cluster. |
| Optional: `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Needed for the Hive client to locate `hive-site.xml` and `core-site.xml`. |
| No explicit script‑level variables are present; any change to the location or database name must be edited directly in the DDL. |

---

## 8. Suggested Improvements (TODO)
1. **Make the DDL idempotent** – prepend `IF NOT EXISTS` and add a pre‑flight check to avoid accidental data loss.  
   ```sql
   CREATE TABLE IF NOT EXISTS mnaas.mnp_portinout_details_reject ( ... );
   ```

2. **Add column comments and table comment** to improve data‑dictionary generation and downstream documentation.  
   ```sql
   `reject_reason` string COMMENT 'Human‑readable reason why the porting transaction was rejected',
   ...
   COMMENT 'Rejected MNP port‑in/out details for audit and re‑processing';
   ```

*(Additional enhancements such as partitioning by `record_insertime` or bucketing by `productid` could be evaluated once data volume justifies it.)*