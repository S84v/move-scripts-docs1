**File:** `mediation-ddls\sim_imei_history.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive table** `mnaas.sim_imei_history` that stores a chronological log of IMEI changes for SIM cards. Each row captures the old and new IMEI values, the associated ICCID/MSISDN/IMSI, timestamps, source system, processing result, and business‑unit metadata. The table points to a static HDFS directory, allowing downstream analytics, audit, and reconciliation jobs to read the change history without moving data into the Hive warehouse.

---

## 2. Core Hive Object(s)

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.sim_imei_history` | **EXTERNAL TABLE** | Persists raw IMEI‑change records as CSV‑delimited text in HDFS. Provides a stable schema for downstream ETL/analytics jobs. |
| `ROW FORMAT SERDE` | Hive SerDe | Parses comma‑separated values using `LazySimpleSerDe`. |
| `STORED AS INPUTFORMAT / OUTPUTFORMAT` | Hadoop I/O classes | Reads plain text files (`TextInputFormat`) and writes using Hive’s ignore‑key text output format. |
| `TBLPROPERTIES` | Metadata | Controls purge behavior, Impala catalog linkage, and audit timestamps. |

*No procedural code (UDFs, stored procedures) is defined in this file.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input Data** | Files located at `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/sim_imei_history` – expected to be CSV rows matching the column order. |
| **Output** | Table metadata registered in the Hive Metastore (and consequently visible to Impala). No data is written by this script; it only registers the external location. |
| **Side Effects** | - Creates/updates the external table definition.<br>- Sets `external.table.purge='true'` → if the table is dropped, underlying HDFS files are deleted.<br>- Updates Impala catalog service IDs and version numbers (via table properties). |
| **Assumptions** | - HDFS path exists and is readable by the Hive/Impala service accounts.<br>- Files are UTF‑8 encoded, newline‑terminated, and use `,` as field delimiter.<br>- Column order in the source files matches the DDL definition.<br>- No partitioning is required for current query patterns. |

---

## 4. Integration Points (How This Table Connects to the Rest of the System)

| Connected Component | Expected Interaction |
|---------------------|----------------------|
| **Downstream ETL jobs** (e.g., `move_sim_inventory_status*`, `msisdn_level_daily_usage_aggr`) | SELECT queries to enrich SIM inventory with IMEI change history for audit or reconciliation. |
| **Impala UI / BI tools** | Direct querying for reporting on IMEI swaps, failure reasons, and source system performance. |
| **Data Quality / Monitoring scripts** | Periodic scans that validate `result` = ‘succeeded’ and flag rows where `reason` is populated. |
| **Data ingestion pipelines** (outside this repo) | Producers that land CSV files into the HDFS location; they rely on the schema defined here to be stable. |
| **Hive Metastore** | Stores the table definition; any schema change must be coordinated with downstream consumers. |

*Because the table is external, any process that writes to the HDFS path must respect the column order and delimiter defined here.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or inaccessible HDFS directory** | Table creation succeeds but queries return no rows or fail with I/O errors. | Validate directory existence & permissions before deployment (e.g., `hdfs dfs -test -d <path>`). |
| **Schema drift** – source files not matching column order or data types (all strings). | Query failures, incorrect data interpretation. | Implement a lightweight pre‑load validation script (e.g., Spark job) that checks column count and basic regex for IMEI format. |
| **External table purge enabled** – accidental `DROP TABLE` removes raw files. | Permanent loss of raw IMEI change logs. | Restrict DROP privileges to a limited admin role; add a procedural guard (e.g., `DROP TABLE IF EXISTS` wrapped in a script that first copies files to a backup location). |
| **Performance degradation** – full table scans on large, unpartitioned data. | Slow downstream jobs. | Consider adding a date partition (e.g., `ingest_date`) if data volume grows; update ingestion pipelines to write into partitioned sub‑folders. |
| **Impala catalog out‑of‑sync** – stale `catalogVersion` after external changes. | Queries return stale schema or missing rows. | Schedule periodic `INVALIDATE METADATA` / `REFRESH` commands in Impala after data loads. |

---

## 6. Running / Debugging the Script

| Step | Command | Purpose |
|------|---------|---------|
| **Create / update table** | `hive -f mediation-ddls/sim_imei_history.hql` | Registers the external table in Hive Metastore. |
| **Verify registration** | `hive -e "DESCRIBE FORMATTED mnaas.sim_imei_history"` | Confirms column definitions, location, and table properties. |
| **Check data visibility** | `hive -e "SELECT COUNT(*) FROM mnaas.sim_imei_history LIMIT 1;"` | Ensures Hive can read the underlying files. |
| **Impala visibility** | `impala-shell -q "SHOW CREATE TABLE mnaas.sim_imei_history;"` | Confirms Impala catalog sync. |
| **Debug missing data** | 1. `hdfs dfs -ls <location>` – verify files exist.<br>2. `hdfs dfs -cat <file> | head` – inspect raw rows.<br>3. Check Hive logs (`/var/log/hive/hive.log`) for parsing errors. |
| **Re‑apply after schema change** | Update the DDL, then run the same `hive -f` command; Impala may need `INVALIDATE METADATA` afterwards. |

*Operators typically trigger this script as part of a nightly schema‑sync job or when a new source system starts writing IMEI change logs.*

---

## 7. External Configuration / Environment Dependencies

| Item | How It Is Used |
|------|----------------|
| **Hive Metastore connection** | Implicit via the `hive` CLI; relies on `hive-site.xml` for JDBC URL, authentication, and warehouse location. |
| **Impala catalog service IDs** (`impala.events.catalogServiceId`, `catalogVersion`) | Populated automatically by Impala; retained in table properties for audit. No manual configuration required. |
| **HDFS Namenode address** (`NN-HA1`) | Hard‑coded in the `LOCATION` clause; must resolve to the active Namenode in the cluster. |
| **Kerberos / Hadoop security** (if enabled) | The Hive/Impala service accounts must have read/write permission on the target HDFS path. |
| **Environment variables** (e.g., `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) | Required by the CLI tools to locate configuration files. |

If the deployment environment changes (e.g., a new Namenode hostname), the `LOCATION` string must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning by Ingestion Date**  
   *Rationale:* Prevent full‑table scans as the log grows; enable efficient pruning for daily/weekly reports.  
   *Action:* Modify DDL to include `ingest_date STRING` (or `DATE`) and `PARTITIONED BY (ingest_date)`. Update ingestion pipelines to write into `.../sim_imei_history/ingest_date=YYYY-MM-DD/`.

2. **Replace String Timestamps with Proper Types**  
   *Rationale:* Enables time‑range queries and proper ordering without string parsing.  
   *Action:* Change `old_timestamp` / `new_timestamp` columns to `TIMESTAMP`. Ensure source files are converted (e.g., via a Spark pre‑processor) before landing in HDFS.

---