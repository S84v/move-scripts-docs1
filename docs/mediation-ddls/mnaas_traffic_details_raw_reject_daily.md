**File:** `mediation-ddls\mnaas_traffic_details_raw_reject_daily.hql`  

---

## 1. High‑Level Summary
This script creates a **Hive external table** named `mnaas.traffic_details_raw_reject_daily`. The table stores daily rejected CDR (Call Detail Record) rows that were originally ingested into the raw traffic feed but failed validation or business rules. Each row contains the full set of raw fields plus a `rejectcode` and `reject_reason`. The table is **partitioned by `partition_date`** (string) and points to an HDFS location where the rejected files are landed. Down‑stream analytics, billing correction, and audit jobs query this table to investigate or re‑process rejected records.

---

## 2. Core Definition (DDL) – Responsibilities

| Element | Responsibility |
|---------|-----------------|
| **CREATE EXTERNAL TABLE `mnaas`.`traffic_details_raw_reject_daily`** | Registers a Hive metastore object that references data stored outside Hive (in HDFS). |
| **Column list (≈ 80 columns)** | Mirrors the schema of the main raw traffic table (`traffic_details_raw_daily`) with two additional reject‑specific columns: `rejectcode` (int) and `reject_reason` (string). |
| **PARTITIONED BY (`partition_date` string)** | Enables daily partition pruning; each day’s reject files are placed under a sub‑directory named after the date. |
| **ROW FORMAT SERDE / INPUTFORMAT / OUTPUTFORMAT** | Uses the default lazy text serde and Hadoop TextInputFormat – expects delimited text files (default Hive delimiter, usually `\001`). |
| **LOCATION** | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_raw_reject_daily` – the physical directory where reject files are dropped by upstream ingestion jobs. |
| **TBLPROPERTIES** | • `external.table.purge='true'` – data is deleted when the table is dropped. <br>• Statistics disabled (`DO_NOT_UPDATE_STATS='true'`). <br>• Impala catalog metadata (serviceId, version, last stats time). <br>• Audit fields (`last_modified_by`, `last_modified_time`). |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Rejected raw CDR files (text, delimited) written by upstream ETL (e.g., `mnaas_traffic_details_raw_daily` ingestion pipeline). <br>- Partition value (`partition_date`) supplied by the producer (usually the processing date). |
| **Outputs** | - Hive/Impala queryable external table exposing the reject records. <br>- Metadata entries in the Hive metastore (table definition, partitions). |
| **Side‑Effects** | - Creation of Hive metastore entries. <br>- If the table is later dropped, all files under the LOCATION are **purged** because of `external.table.purge='true'`. |
| **Assumptions** | - HDFS path exists and is writable by the ingestion user. <br>- Files conform to the column order and delimiter expected by the LazySimpleSerDe. <br>- `partition_date` is provided as a string matching the directory naming convention (`partition_date=YYYY-MM-DD`). <br>- No schema evolution is required after table creation (columns are static). |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | Interaction |
|---------------------|-------------|
| **`mnaas_traffic_details_raw_daily.hql`** | Shares the same column set; reject table is a “sink” for rows that fail validation in the raw‑daily pipeline. |
| **Ingestion/Validator Jobs** (e.g., Spark/MapReduce jobs that read the raw daily feed) | Write rejected rows to the HDFS location defined above, using the same partition layout. |
| **Down‑stream Processing** (e.g., `mnaas_traffic_details_billing.hql`, audit scripts) | Query this table to generate rejection reports, trigger re‑processing, or feed error‑handling dashboards. |
| **Impala / Hive UI** | Users run `SELECT * FROM mnaas.traffic_details_raw_reject_daily WHERE partition_date='2024‑11‑30'` to investigate issues. |
| **Metadata / Scheduler** (e.g., Airflow, Oozie) | May run `MSCK REPAIR TABLE mnaas.traffic_details_raw_reject_daily` after new partitions are added, or issue `ALTER TABLE … ADD PARTITION`. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Schema drift** – upstream changes add/remove columns but the reject table is not updated. | Version‑control DDLs; run a nightly schema‑compare job; automate `ALTER TABLE … REPLACE COLUMNS` when a change is approved. |
| **Orphaned partitions** – partitions created but never populated, leading to storage bloat. | Periodic housekeeping job that drops empty partitions (`ALTER TABLE … DROP IF EXISTS PARTITION`). |
| **Data loss on drop** – `external.table.purge='true'` will delete files if the table is accidentally dropped. | Enforce RBAC so only privileged users can DROP; keep a backup of the HDFS directory (snapshot or replication). |
| **Incorrect delimiter / malformed rows** – cause query failures or truncated data. | Validate incoming files with a lightweight parser before moving them to the location; log and quarantine malformed files. |
| **Permission mismatch** – ingestion user cannot write to the HDFS location. | Standardize a Hadoop ACL / POSIX permission set (`chmod 770`, group `hive`) and document the required user/group. |

---

## 6. Running / Debugging the Table

1. **Validate Table Exists**  
   ```sql
   SHOW CREATE TABLE mnaas.traffic_details_raw_reject_daily;
   ```

2. **Inspect Partitions**  
   ```sql
   SHOW PARTITIONS mnaas.traffic_details_raw_reject_daily;
   ```

3. **Sample Query** (verify data shape)  
   ```sql
   SELECT partition_date, cdrid, rejectcode, reject_reason
   FROM mnaas.traffic_details_raw_reject_daily
   WHERE partition_date = '2024-11-30'
   LIMIT 10;
   ```

4. **Repair Missing Partitions** (if a new date folder appears)  
   ```sql
   MSCK REPAIR TABLE mnaas.traffic_details_raw_reject_daily;
   ```

5. **Check File Layout** (via HDFS)  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/traffic_details_raw_reject_daily/partition_date=2024-11-30/
   ```

6. **Debug Rejection Logic**  
   - Locate the upstream job that writes the reject files (usually a Spark/MapReduce job).  
   - Review its logs for the `rejectcode` mapping to business rules.  

---

## 7. External Config / Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` URI; may be templated via a deployment script that injects the active HA namenode. |
| Hive Metastore connection (`hive.metastore.uris`) | Required for the DDL to register the table. |
| Hadoop user / group (`HADOOP_USER_NAME`, `HADOOP_GROUP`) | Determines write permissions to the HDFS location. |
| Impala catalog service ID / version (shown in `TBLPROPERTIES`) | Managed automatically by Impala; not manually edited. |
| Partition date format (e.g., `YYYY-MM-DD`) | Expected by downstream jobs; usually derived from an environment variable like `PROCESS_DATE`. |

If any of these values are externalised (e.g., via a `*.properties` file or CI/CD template), the deployment pipeline should substitute them before executing the DDL.

---

## 8. Suggested TODO / Improvements

1. **Add a `reject_timestamp` column** (type `timestamp`) to capture the exact moment a record was classified as rejected. This aids audit trails and latency monitoring.  
2. **Implement a schema‑evolution guard**: create a lightweight CI job that parses the DDL and compares it against the current Hive metastore definition; fail the build if columns diverge.  

---