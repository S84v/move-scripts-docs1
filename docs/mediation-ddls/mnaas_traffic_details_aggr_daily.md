**File:** `mediation-ddls\mnaas_traffic_details_aggr_daily.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive/Impala table** named `traffic_details_aggr_daily` in the `mnaas` database. The table stores daily‑aggregated traffic metrics (voice, SMS, data) per customer‑derived identifiers, enriched with business‑unit, geographic, and commercial‑offer attributes. Data is persisted as **Parquet files** under a fixed HDFS location and is partitioned by `partition_date` (string). The table is intended to be populated by upstream ETL jobs that roll‑up raw traffic records (e.g., `mnaas_traffic_Details_raw_daily`) and to serve downstream reporting scripts such as `mnaas_ra_traffic_details_summary.hql`.

---

## 2. Core Objects Defined

| Object | Responsibility |
|--------|-----------------|
| **Table `mnaas.traffic_details_aggr_daily`** | Holds daily aggregated traffic metrics; external so the underlying files are managed outside the Hive Metastore. |
| **Columns** (e.g., `tcl_secs_id`, `serv_abbr`, `businessunit`, … `regn_nm`) | Capture identifiers, business context, traffic direction, event type, and aggregated numeric measures (counts, durations, bytes, charges). |
| **Partition `partition_date` (string)** | Enables efficient pruning for date‑specific queries; aligns with daily load cycles. |
| **ROW FORMAT / SERDE** | Uses `ParquetHiveSerDe` for columnar storage; serialization properties are set but largely ignored for Parquet. |
| **STORED AS INPUTFORMAT / OUTPUTFORMAT** | Explicitly points to Parquet MapReduce input/output formats. |
| **LOCATION** | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_aggr_daily` – the physical directory where daily Parquet files are written. |
| **TBLPROPERTIES** | Controls statistics handling, purge behavior, and Impala catalog metadata (e.g., `external.table.purge='true'`). |

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | - Parquet files written by upstream aggregation jobs (typically a Spark/MapReduce job that reads `mnaas_traffic_Details_raw_daily`).<br>- HDFS namespace `NN-HA1` must be reachable and writable by the job user. |
| **Outputs** | - Hive/Impala metadata entry for the external table.<br>- Queryable view of the aggregated data for downstream reporting. |
| **Side Effects** | - Registers the external table in the Hive Metastore.<br>- Because `external.table.purge='true'`, dropping the table will also delete the underlying files (non‑standard for external tables). |
| **Assumptions** | - The HDFS path exists and has appropriate permissions (read/write for the ETL user).<br>- Impala/Hive versions support the specified Parquet SerDe and input/output formats.<br>- Partition values are supplied correctly by the load process (matching `partition_date`). |

---

## 4. Integration Points

| Direction | Connected Component | Interaction |
|-----------|---------------------|-------------|
| **Upstream** | `mnaas_traffic_Details_raw_daily.hql` (raw daily traffic) | Aggregation job reads raw data, computes daily sums/counts, writes Parquet files into the table’s location, then adds the appropriate `partition_date` partition. |
| **Downstream** | `mnaas_ra_traffic_details_summary.hql` (report) | Queries this table (often with `WHERE partition_date = …`) to produce business‑unit or commercial‑offer summaries. |
| **Other** | `mnaas_ra_msisdn_wise_usage_report.hql`, `mnaas_ra_ic_calldate_monthly_rep.hql` | May join on common keys (`tcl_secs_id`, `serv_abbr`, `country`) to enrich usage reports. |
| **Catalog** | Impala catalog service (ID `b9c115786e934562:94a82089945b48d6`) | The table metadata is propagated to Impala for low‑latency SQL access. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Partition drift** – load job writes files with an unexpected `partition_date` value. | Queries miss data; stale partitions accumulate. | Enforce partition naming convention in the ETL job; add a validation step that lists partitions after load. |
| **Location change** – HDFS path altered (e.g., NN rename) without updating DDL. | Table becomes inaccessible; downstream jobs fail. | Externalize the location into a configuration property (e.g., `${TRAFFIC_AGGR_DAILY_PATH}`) and reference it in the script. |
| **Permission issues** – user lacks write access to the external directory. | Load job aborts; table remains empty. | Include a pre‑run check (`hdfs dfs -test -w <path>`) and alert on failure. |
| **Statistics not collected** – `DO_NOT_UPDATE_STATS='true'` disables automatic stats, leading to poor query plans. | Slow reporting queries. | Schedule a periodic `ANALYZE TABLE traffic_details_aggr_daily COMPUTE STATISTICS` (or Impala `COMPUTE INCREMENTAL STATS`). |
| **Accidental purge** – dropping the table removes raw Parquet files due to `external.table.purge='true'`. | Data loss. | Restrict DROP privileges; consider setting `external.table.purge='false'` unless intentional. |

---

## 6. Running / Debugging the Script

1. **Execution** (as part of a deployment pipeline or manually):  
   ```bash
   hive -f mediation-ddls/mnaas_traffic_details_aggr_daily.hql
   # or, for Impala:
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_traffic_details_aggr_daily.hql
   ```

2. **Verification** after run:  
   ```sql
   SHOW CREATE TABLE mnaas.traffic_details_aggr_daily;
   SHOW PARTITIONS mnaas.traffic_details_aggr_daily;
   SELECT COUNT(*) FROM mnaas.traffic_details_aggr_daily LIMIT 10;
   ```

3. **Debugging Tips**  
   - If the table already exists, the script will fail. Add `IF NOT EXISTS` or drop/recreate as needed.  
   - Check the Hive Metastore logs for permission or path errors.  
   - Verify the underlying HDFS directory with `hdfs dfs -ls <location>`.  
   - Confirm that the Impala catalog refreshed (`INVALIDATE METADATA mnaas.traffic_details_aggr_daily;`).  

---

## 7. External Configuration & Environment Variables

| Item | Current Usage | Suggested Improvement |
|------|---------------|-----------------------|
| **HDFS Namenode** (`NN-HA1`) | Hard‑coded in the `LOCATION` clause. | Replace with a variable (e.g., `${HDFS_NN}`) sourced from a central config file or environment. |
| **Database name** (`mnaas`) | Fixed in the script. | Parameterize if the same DDL is reused across environments (dev/test/prod). |
| **Table properties** (stats flags) | Static values. | Consider toggling via a config flag to enable/disable stats collection per environment. |

---

## 8. Suggested TODO / Improvements

1. **Make the script idempotent** – prepend `CREATE EXTERNAL TABLE IF NOT EXISTS` and add a conditional `DROP TABLE IF EXISTS` block guarded by a deployment flag.  
2. **Automate statistics collection** – append an `ANALYZE TABLE` statement (or Impala `COMPUTE INCREMENTAL STATS`) at the end of the script, optionally gated by a config switch, to ensure query planners have up‑to‑date cardinality information.  

---