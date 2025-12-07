**File:** `mediation-ddls/traffic_details_raw_daily_with_no_dups.hql`  
**Type:** Hive DDL – creates an **external** Parquet table `mnaas.traffic_details_raw_daily_with_no_dups`.

---

## 1. High‑Level Summary
This script defines the schema, storage format, and partitioning for the raw, de‑duplicated daily traffic CDR (Call Detail Record) data used throughout the mediation layer. The table is external, pointing to a fixed HDFS location, and is partitioned by `partition_date` (string). Down‑stream aggregation, billing, and analytics jobs (e.g., `traffic_aggr_adhoc`, `msisdn_level_daily_usage_aggr`, `traffic_details_raw_daily`) read from this table; upstream ingestion pipelines write Parquet files into the same HDFS path after deduplication.

---

## 2. Core Artifact(s)

| Artifact | Responsibility |
|----------|-----------------|
| **Table `mnaas.traffic_details_raw_daily_with_no_dups`** | Holds one‑day‑granular, de‑duplicated CDR rows with > 100 columns covering identifiers, billing, usage metrics, device info, and derived fields. |
| **Partition `partition_date`** | Enables daily pruning; each day’s data lives in a sub‑directory `<...>/partition_date=YYYYMMDD/`. |
| **SerDe / Input‑Output Formats** | Parquet (`MapredParquetInputFormat/OutputFormat`) with Hive‑provided Parquet SerDe. |
| **Table Properties** | Controls stats handling, external purge, Impala catalog sync, Spark schema propagation, etc. |

*No procedural code (functions, classes) is present – the file is purely declarative.*

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Input data source** | Up‑stream ETL job that reads raw CDR files (e.g., from S3, FTP, or a streaming source), performs deduplication, transforms to the column layout, and writes Parquet files to the HDFS location `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_raw_daily_with_no_dups/partition_date=YYYYMMDD/`. |
| **Outputs** | Hive/Impala external table metadata; downstream queries see the data as a regular table. No physical data movement occurs at DDL time. |
| **Side effects** | - Registers the external table in the Hive metastore.<br>- Updates Impala catalog (via `impala.events.catalogServiceId` etc.).<br>- Sets `external.table.purge='true'` meaning that dropping the table will also delete the underlying HDFS files. |
| **Assumptions** | - Data written to the location conforms exactly to the declared schema (Parquet column order, types).<br>- Partition column `partition_date` is always supplied (string, e.g., `20231130`).<br>- No further DDL changes are made without coordinating with downstream jobs that rely on column positions. |

---

## 4. Integration Points (How it Connects to Other Scripts)

| Connected Component | Relationship |
|---------------------|--------------|
| **`traffic_aggr_adhoc.hql` / `traffic_aggr_adhoc_view.hql`** | These scripts create aggregation tables/views that **SELECT** from `traffic_details_raw_daily_with_no_dups`. |
| **`msisdn_level_daily_usage_aggr.hql`** | Consumes the raw usage columns (`data_usage_in_kb`, `voice_usage_in_mins`, `sms_usage`) from this table for per‑MSISDN daily roll‑ups. |
| **`traffic_details_raw_daily.hql` (if exists)** | May be a predecessor that loads raw data *with* duplicates; this table is the deduped version used for production billing. |
| **Ingestion pipelines (Shell/Scala/Python jobs)** | Write Parquet files into the HDFS location; they reference the same `partition_date` naming convention. |
| **Reporting / BI tools (e.g., Tableau, PowerBI via Impala)** | Query this table directly for ad‑hoc analysis. |
| **Data quality / validation jobs** | Run `SELECT COUNT(*)`, `SELECT DISTINCT rejectcode` etc. against this table to verify that deduplication succeeded. |

*All downstream jobs assume the table exists and is refreshed daily (new partition added).*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream writer adds/renames a column without updating DDL. | Queries may fail or return wrong results. | Enforce schema validation in the ingestion job (e.g., Avro/Parquet schema check) and version‑control DDL changes with CI gate. |
| **Stale partitions** – missing daily partition due to failed load. | Downstream aggregations produce gaps. | Implement a daily partition‑audit job that checks for expected `partition_date` directories and raises alerts. |
| **External table purge** – accidental `DROP TABLE` removes raw data. | Irrecoverable data loss. | Restrict DROP privileges; use `DROP TABLE IF EXISTS` only in controlled maintenance windows; enable HDFS snapshots for the location. |
| **Statistics not refreshed** (`DO_NOT_UPDATE_STATS='true'`). | Query planners may choose sub‑optimal execution plans. | Schedule `ANALYZE TABLE … COMPUTE STATISTICS` after each partition load, or set `DO_NOT_UPDATE_STATS='false'` if stats are needed. |
| **Large partition size** – a single day may contain many GBs, causing long scans. | Performance degradation. | Consider further sub‑partitioning (e.g., by `companyintid` or `calltype`) if query patterns justify it. |
| **Incorrect `partition_date` format** – string vs. date type. | Partition pruning fails. | Standardize on `YYYYMMDD` string; add validation step in the loader. |

---

## 6. Running / Debugging the Script

| Action | Command | Notes |
|--------|---------|-------|
| **Create/Replace the table** | `hive -f traffic_details_raw_daily_with_no_dups.hql` <br>or <br>`impala-shell -i <impala-host> -f traffic_details_raw_daily_with_no_dups.hql` | The script uses `CREATE EXTERNAL TABLE`; re‑run only if schema changes. |
| **Verify table definition** | `SHOW CREATE TABLE mnaas.traffic_details_raw_daily_with_no_dups;` | Compare output to source file for drift. |
| **Inspect partitions** | `SHOW PARTITIONS mnaas.traffic_details_raw_daily_with_no_dups;` | Ensure the expected `partition_date` appears. |
| **Sample data** | `SELECT * FROM mnaas.traffic_details_raw_daily_with_no_dups LIMIT 10;` | Quick sanity check of column values. |
| **Check file location** | `hdfs dfs -ls /user/hive/warehouse/mnaas.db/traffic_details_raw_daily_with_no_dups/partition_date=20231130/` | Verify that Parquet files exist and are readable. |
| **Refresh Impala metadata** (if using Impala) | `INVALIDATE METADATA mnaas.traffic_details_raw_daily_with_no_dups;` | Needed after adding new partitions. |
| **Debug missing columns** | Run `DESCRIBE FORMATTED mnaas.traffic_details_raw_daily_with_no_dups;` | Confirms column types and comments. |

---

## 7. External Config / Environment Dependencies

| Item | Usage |
|------|-------|
| **Hive Metastore URI** (`hive.metastore.uris`) | Required for Hive to register the external table. |
| **HDFS Namenode address** (`NN-HA1`) | Hard‑coded in the `LOCATION` clause; any change to the cluster topology must be reflected in the DDL. |
| **Impala catalog service ID** (`impala.events.catalogServiceId`) | Auto‑generated; not manually set but may be referenced by downstream Impala jobs. |
| **Spark‑SQL compatibility** (`spark.sql.create.version`) | Indicates the table was created with Spark 2.2‑compatible DDL; downstream Spark jobs should respect this version. |
| **Environment variables for ingestion** (e.g., `RAW_CDR_PATH`, `DEDUPED_OUTPUT_PATH`) | Not referenced directly in this file but used by the upstream loader that writes to the `LOCATION`. |

---

## 8. Suggested TODO / Improvements

1. **Automate Statistics Refresh** – Add a post‑load step (e.g., `ANALYZE TABLE mnaas.traffic_details_raw_daily_with_no_dups PARTITION (partition_date='${date}') COMPUTE STATISTICS;`) to keep the query planner optimal.
2. **Introduce a Date‑typed Partition** – Change `partition_date` from `string` to `date` (or `int` `yyyymmdd`) to enable proper date functions and partition pruning in Hive/Impala. This will require a migration script that rewrites existing partitions.  

--- 

*End of documentation.*