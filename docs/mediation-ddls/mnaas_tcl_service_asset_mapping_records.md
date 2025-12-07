**File:** `mediation-ddls\mnaas_tcl_service_asset_mapping_records.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the **`tcl_service_asset_mapping_records`** table in the `mnaas` database. The table stores a one‑to‑many mapping between telecom service assets (e.g., SIM/ICCID, VIN, device type) and the logical services they belong to, together with lifecycle timestamps and status flags. It is partitioned by `partition_month` and `datasource`, stored as ORC, and defined as an *insert‑only transactional* table, enabling downstream ETL jobs to append records safely while supporting ACID‑style reads.

---

## 2. Core Definition (Responsibilities)

| Element | Responsibility / Description |
|---------|------------------------------|
| **Table name** `mnaas.tcl_service_asset_mapping_records` | Central repository for service‑to‑asset mapping records used by downstream reporting and validation jobs. |
| **Columns** | <ul><li>`secsid` – internal service identifier (string)</li><li>`customername` – human‑readable customer name</li><li>`vinid` – vehicle identification number (if applicable)</li><li>`devicetype` – type of device (e.g., “SIM”, “eSIM”)</li><li>`countryofsale` – ISO country code where the asset was sold</li><li>`iccid` – SIM identifier</li><li>`eid` – eUICC identifier</li><li>`servicename` – logical service name (e.g., “DataPlanA”)</li><li>`service_start_date` / `service_end_data` – lifecycle timestamps</li><li>`iccid_status` – current status of the ICCID (active, suspended, etc.)</li><li>`iccid_activation_date` / `iccid_last_update_date` – audit timestamps for the ICCID</li><li>`record_insert_time` – ingestion timestamp (used for CDC / debugging)</li><li>`data_reported` – raw payload or flag indicating source reporting status</li><li>`service_status` – overall status of the service‑asset link</li></ul> |
| **Partitions** | `partition_month` (string, e.g., “202312”) – enables time‑based pruning; `datasource` (string) – distinguishes source system (e.g., “Geneva”, “MNO‑X”). |
| **Storage format** | ORC (`OrcSerde`, `OrcInputFormat`, `OrcOutputFormat`) – columnar compression, predicate push‑down. |
| **Location** | `hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/tcl_service_asset_mapping_records` – managed Hive directory on HDFS. |
| **Table properties** | `transactional='true'`, `transactional_properties='insert_only'` – allows ACID‑style appends without updates; `bucketing_version='2'` (future‑proofing). |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | None at DDL execution time. The script assumes the Hive metastore is reachable and the HDFS namenode `NN-HA1` is accessible. |
| **Outputs** | - A new Hive table definition stored in the metastore.<br>- A managed HDFS directory created at the `LOCATION` path (empty until data is inserted). |
| **Side‑effects** | - May overwrite an existing table if `DROP TABLE IF EXISTS` is added upstream (not present here).<br>- Consumes HDFS namespace; partitions will create sub‑directories under the location. |
| **Assumptions** | - Hive version ≥ 2.3 (supports insert‑only transactional tables).<br>- ORC libraries are installed on the cluster.<br>- The `mnaas` database already exists.<br>- Proper HDFS permissions for the Hive service user to create directories under the given path. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **ETL ingestion jobs** (e.g., `mnaas_*_rep` scripts) | Append records into this table using `INSERT INTO mnaas.tcl_service_asset_mapping_records PARTITION (partition_month='202312', datasource='Geneva') SELECT …`. |
| **Reporting / analytics jobs** (e.g., `mnaas_tcl_asset_pkg_mapping.hql`, `mnaas_msisdn_wise_usage_report.hql`) | Join on `iccid`, `eid`, or `secsid` to enrich usage or billing data. |
| **Data quality / reconciliation scripts** | Query `record_insert_time` and `data_reported` to detect missing or duplicate mappings. |
| **Scheduler / workflow engine** (e.g., Oozie, Airflow) | The DDL is typically run once during environment provisioning or as part of a “schema migration” DAG. |
| **External configuration** | The HDFS namenode address (`NN-HA1`) is often injected via an environment variable or a Hive site property (`fs.defaultFS`). The script itself does not reference any variable, but the execution environment must resolve that hostname. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Table already exists with incompatible schema** | Job failures, data loss if overwritten. | Add `IF NOT EXISTS` guard or version‑controlled migrations (e.g., using HiveSchemaTool). |
| **Incorrect partition values** (e.g., typo in `partition_month`) | Data lands in wrong partition, breaking downstream time‑based queries. | Enforce partition naming conventions in ingestion code; add validation step before `INSERT`. |
| **Insufficient HDFS storage / quota** | Job stalls during insert, causing back‑pressure. | Monitor HDFS usage; set alerts on the table’s directory size. |
| **Transactional insert‑only property not supported on older Hive** | Table creation fails or behaves non‑transactionally. | Verify Hive version before deployment; keep a compatibility matrix. |
| **Permission issues on HDFS location** | Table creation fails; downstream jobs cannot write. | Ensure Hive service user has `WRITE` permission on the target path; test with a dry‑run. |
| **Schema drift (new columns added elsewhere)** | Queries that expect older schema break. | Use a schema‑evolution policy; keep DDL under version control and run migration scripts when adding columns. |

---

## 6. Running / Debugging the Script

1. **Typical execution command** (from a CI/CD or scheduler step):  
   ```bash
   hive -f mediation-ddls/mnaas_tcl_service_asset_mapping_records.hql
   ```
   *or* using Beeline with a JDBC URL:
   ```bash
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaas_tcl_service_asset_mapping_records.hql
   ```

2. **Verification steps** after execution:  
   ```sql
   SHOW CREATE TABLE mnaas.tcl_service_asset_mapping_records;
   DESCRIBE FORMATTED mnaas.tcl_service_asset_mapping_records;
   ```

3. **Debugging tips**  
   - Check HiveServer2 logs (`/var/log/hive/hiveserver2.log`) for syntax errors.  
   - Verify the HDFS directory exists: `hdfs dfs -ls /warehouse/tablespace/managed/hive/mnaas.db/tcl_service_asset_mapping_records`.  
   - If the table is not created, confirm that the `mnaas` database exists: `SHOW DATABASES LIKE 'mnaas';`.  
   - Use `hive -e "SELECT COUNT(*) FROM mnaas.tcl_service_asset_mapping_records;"` to confirm it is empty (expected after creation).  

---

## 7. External Configurations / Environment Variables Referenced

| Item | Usage |
|------|-------|
| `NN-HA1` (HDFS namenode hostname) | Hard‑coded in the `LOCATION` clause; must resolve in the cluster’s DNS or `/etc/hosts`. |
| Hive metastore connection (`hive.metastore.uris`) | Not in the script but required for the Hive client to register the table. |
| Hadoop user (`HADOOP_USER_NAME`) | Determines the HDFS permissions for the created directory. |
| Optional: `HIVE_CONF_DIR` / `HIVE_OPTS` | May contain custom serde or ORC settings that affect table creation. |

If the environment uses a templating layer (e.g., Jinja, Velocity) to inject the namenode, the script would be rendered before execution; verify the rendering step in the CI pipeline.

---

## 8. Suggested Improvements (TODO)

1. **Add idempotent guard** – prepend the DDL with `CREATE TABLE IF NOT EXISTS` or wrap in a migration script that checks for existing schema version.  
2. **Externalize the HDFS location** – replace the hard‑coded `NN-HA1` with a variable (e.g., `${hdfs.base.path}`) sourced from a central config file, making the script portable across dev / prod clusters.  

---