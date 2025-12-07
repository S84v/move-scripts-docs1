**File:** `mediation-ddls/traffic_details_raw_daily_with_no_dups_SMS_ONLY.hql`  
**Type:** Hive DDL – creates an **external** Parquet table `mnaas.traffic_details_raw_daily_with_no_dups_sms_only`.

---

## 1. High‑Level Summary
This script defines an external Hive table that stores raw, de‑duplicated SMS‑only traffic records on a daily basis. The table is partition‑ed by `partition_date` and points to a fixed HDFS location under the `mnaas` database. Down‑stream aggregation, reporting, and mediation jobs (e.g., `traffic_aggr_adhoc`, `traffic_details_raw_daily_with_no_dups`) read from this table to compute usage metrics, charge‑back calculations, and regulatory extracts.

---

## 2. Core Artifact(s)

| Artifact | Responsibility |
|----------|-----------------|
| **`traffic_details_raw_daily_with_no_dups_sms_only` (Hive external table)** | Holds raw SMS event rows (one row per SMS) with full context (business unit, network identifiers, subscriber IDs, timestamps, usage amount). The table is **external** – Hive only registers metadata; the underlying Parquet files are managed outside Hive. |
| **DDL statement (`CREATE EXTERNAL TABLE …`)** | Creates the table definition, column list, partitioning, SerDe, storage format, location, and table properties. No procedural code or functions are defined in this file. |

*No procedural classes or functions are present; the file is pure DDL.*

---

## 3. Interfaces

| Category | Details |
|----------|---------|
| **Inputs (external to Hive)** | • HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_raw_daily_with_no_dups_SMS_ONLY` – must exist and be writable by the Hive/Impala service accounts.<br>• Daily data files (Parquet) placed into the appropriate `partition_date=` sub‑directory by upstream ingestion pipelines (e.g., file‑drop, Spark jobs, or legacy ETL). |
| **Outputs** | • Hive metadata entry for the external table.<br>• Queryable view of the Parquet files for downstream jobs (e.g., `SELECT … FROM mnaas.traffic_details_raw_daily_with_no_dups_sms_only WHERE partition_date='2023‑09‑01'`). |
| **Side Effects** | • Registers the table in the Hive metastore.<br>• Sets table properties (`external.table.purge='true'`, `DO_NOT_UPDATE_STATS='true'`, etc.). |
| **Assumptions** | • The HDFS namenode address (`NN-HA1`) is reachable from Hive/Impala.<br>• The directory hierarchy follows the Hive warehouse convention (`.../mnaas.db/...`).<br>• Data files conform to the declared schema (Parquet column types match the DDL).<br>• Partition values are supplied as strings matching the `partition_date` format used by upstream pipelines (typically `yyyy-MM-dd`). |
| **External Services** | • Hive Metastore (for DDL registration).<br>• Impala catalog (properties reference Impala catalog IDs).<br>• HDFS (storage). |

---

## 4. Integration Points

| Connected Component | Relationship |
|----------------------|--------------|
| **`traffic_details_raw_daily_with_no_dups.hql`** | Companion table that stores *all* traffic (voice, data, SMS). Down‑stream jobs may UNION or JOIN the SMS‑only table with the generic table for specialized SMS analytics. |
| **`traffic_aggr_adhoc.hql` / `traffic_aggr_adhoc_view.hql`** | Aggregation scripts that read from this table (via `WHERE calltype='sms'` or directly referencing the SMS‑only table) to produce daily/weekly/monthly usage summaries. |
| **Ingestion pipelines** (e.g., Spark, Flume, custom “move” scripts) | Write Parquet files into the HDFS location, partitioned by `partition_date`. Those pipelines must invoke `MSCK REPAIR TABLE` or `ALTER TABLE … ADD PARTITION` after each load. |
| **Reporting / Billing jobs** | Query this table to calculate SMS usage per subscriber, per business unit, or per commercial offer. |
| **Data‑quality / de‑duplication jobs** | The “no_dups” qualifier indicates that upstream processes have already removed duplicate SMS events; downstream jobs assume uniqueness. |

*If the repository contains a master orchestration script (e.g., a Bash/Airflow DAG) that runs all DDL files in sequence, this file would be executed after the generic `traffic_details_raw_daily_with_no_dups` table is created.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale or missing HDFS directory** | Table appears but queries return no rows → downstream billing errors. | Automate a pre‑flight check that the HDFS path exists and is writable; alert if missing. |
| **Partition drift** (partition value format mismatch) | Data lands in wrong partition or is invisible to queries. | Enforce a strict naming convention (`yyyy-MM-dd`) in upstream pipelines; run `MSCK REPAIR TABLE` after each load. |
| **Schema mismatch** (upstream writes a column with a different type) | Query failures, corrupted results. | Version the table schema (e.g., include a `schema_version` column) and add a CI test that validates Parquet schema against the DDL. |
| **External table purge flag** (`external.table.purge='true'`) | Deleting a partition via `DROP PARTITION` will delete underlying files – accidental data loss. | Restrict DROP/ALTER privileges to a limited admin group; require a change‑control ticket for partition deletions. |
| **Impala catalog desynchronisation** (catalog IDs in `TBLPROPERTIES`) | Impala may serve stale metadata, causing query errors. | Schedule a periodic `INVALIDATE METADATA` or `REFRESH` on the table; monitor Impala catalog logs for mismatches. |
| **Permissions** (Hive/Impala service accounts lack HDFS rights) | Table creation succeeds but data cannot be read/written. | Verify ACLs on the target HDFS directory (`chmod 770`, `setfacl`) for the Hive/Impala service principals. |

---

## 6. Execution & Debugging Guide

1. **Run the DDL**  
   ```bash
   hive -f mediation-ddls/traffic_details_raw_daily_with_no_dups_SMS_ONLY.hql
   # or, for Impala
   impala-shell -i <impala-daemon> -f traffic_details_raw_daily_with_no_dups_SMS_ONLY.hql
   ```

2. **Validate creation**  
   ```sql
   SHOW CREATE TABLE mnaas.traffic_details_raw_daily_with_no_dups_sms_only;
   DESCRIBE FORMATTED mnaas.traffic_details_raw_daily_with_no_dups_sms_only;
   ```

3. **Check HDFS location**  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/traffic_details_raw_daily_with_no_dups_SMS_ONLY
   ```

4. **Add a test partition** (simulate upstream load)  
   ```sql
   ALTER TABLE mnaas.traffic_details_raw_daily_with_no_dups_sms_only
     ADD IF NOT EXISTS PARTITION (partition_date='2023-09-01')
     LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_raw_daily_with_no_dups_SMS_ONLY/partition_date=2023-09-01';
   ```

5. **Query sample rows**  
   ```sql
   SELECT msisdn, calltimestamp, sms_usage
   FROM mnaas.traffic_details_raw_daily_with_no_dups_sms_only
   WHERE partition_date='2023-09-01' LIMIT 10;
   ```

6. **Debugging tips**  
   * If the table is visible but empty, verify that upstream pipelines placed Parquet files under the correct partition directory.  
   * If you see “Invalid schema” errors, run `parquet-tools schema <file>` on a sample file and compare to the DDL.  
   * For Impala metadata issues, run `INVALIDATE METADATA mnaas.traffic_details_raw_daily_with_no_dups_sms_only;` then re‑query.

---

## 7. Configuration / Environment Dependencies

| Item | How It Is Used |
|------|----------------|
| **HDFS namenode address (`NN-HA1`)** | Hard‑coded in the `LOCATION` clause; must resolve via the cluster’s DNS or `/etc/hosts`. |
| **Hive Metastore URI** | Implicit – the Hive client reads the metastore configuration from `hive-site.xml`. |
| **Impala catalog IDs** (`impala.events.catalogServiceId`, `catalogVersion`) | Populated automatically by Impala when the table is created; retained for catalog synchronization. |
| **Warehouse root (`/user/hive/warehouse`)** | Standard Hive warehouse path; any change to `hive.metastore.warehouse.dir` would require updating the `LOCATION`. |
| **Environment variables** (e.g., `HADOOP_CONF_DIR`, `HIVE_CONF_DIR`) | Needed by the Hive/Impala client to locate configuration files; not referenced directly in the script. |

If the deployment uses parameterised DDL (e.g., `${WAREHOUSE_ROOT}`), this file currently does **not** – the path is static. Consider externalising it if the warehouse location changes across environments (dev/test/prod).

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard** to make the script idempotent:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.traffic_details_raw_daily_with_no_dups_sms_only ( … )
   ```

2. **Automate partition discovery** by appending a post‑creation step:  
   ```sql
   MSCK REPAIR TABLE mnaas.traffic_details_raw_daily_with_no_dups_sms_only;
   ```
   This ensures that any partitions added by upstream jobs become immediately visible without manual `ALTER TABLE … ADD PARTITION` calls.

*Optional*: Externalise the HDFS location into a Hive configuration variable (e.g., `${traffic.sms.only.path}`) to simplify environment‑specific deployments.