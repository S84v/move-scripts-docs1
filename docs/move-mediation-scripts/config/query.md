**High‑Level Documentation – `move-mediation-scripts/config/query.yml`**

---

### 1. Purpose (one‑paragraph summary)
`query.yml` is the central catalogue of Hive/Impala SQL statements used throughout the MOVE mediation platform. Each entry maps a logical name (e.g., `traffic_details_hol`, `kyc_feed_load`) to a concrete INSERT/SELECT/TRUNCATE statement that populates, refreshes, or aggregates data in the `mnaas` data‑warehouse. The file is read by the various Bash/Python wrappers (e.g., `traffic_nodups_summation.py`, `startTestMoveConsumer.sh`, `traffic_aggr_adhoc_refresh.sh`) which substitute runtime parameters (dates, table names, partition keys) and execute the queries via `spark‑submit`, `hive -e`, or `impala-shell`. By externalising the SQL, the system can evolve data‑model changes without touching the orchestration scripts.

---

### 2. Important “Classes” – Named Queries & Responsibilities  

| Query Name | Responsibility | Key Tables / Targets |
|------------|----------------|----------------------|
| **traffic_details_hol** / **traffic_details_hol_20Aug25** | Load raw HOL (Home‑Office‑Line) traffic records into `traffic_details_raw_daily` after enriching with KYC & SECSID mapping. | Source: `traffic_details_inter_reject_raw_daily`; Joins: `kyc_snapshot`, `secsid_parent_child_mapping`. |
| **traffic_details_sng** | Same as above for Singapore‑origin files (`SNG`). | Source: `traffic_details_inter_reject_raw_daily`; Right‑outer join with `feed_customer_secs_mapping`. |
| **siriusxm_reject_query** | Persist rejected SiriusXM CDRs into `siriusxm_reject_data`. | Source: `traffic_details_inter_raw_daily`. |
| **siriusxm_reProcessingQuery** | Re‑process rejected SiriusXM records after KYC match. | Sources: `kyc_snapshot`, `siriusxm_reject_data`, `secsid_parent_child_mapping`. |
| **activations_hol / activations_sng** | Load activation records (HOL & SNG) into `activations_raw_daily`. | Source: `activations_inter_raw_daily`. |
| **actives_hol / actives_sng** | Load active‑SIM records into `actives_raw_daily`. | Source: `actives_inter_raw_daily`. |
| **tolling_hol / tolling_sng** | Load tolling CDRs into `tolling_raw_daily`. | Source: `tolling_inter_raw_daily`. |
| **siminventory**, **TapErrors**, **FailedEvents**, **IMEIChange_feed**, **IPVProbe_feed** | Generic “feed‑to‑table” loaders that copy raw files into destination tables, applying a partition‑date derived from the filename or event timestamp. |
| **kyc_feed_truncate_** | Truncate the staging table `kyc_mapping_snapshot` before a fresh load. |
| **kyc_feed_load** | Insert distinct KYC rows from `kyc_inter_raw_daily` into the canonical `kyc_iccid_wise_country_mapping`. |
| **kyc_feed_snapshot** | Build the latest‑per‑ICCID view `kyc_snapshot` (most recent KYC record). |
| **kyc_feed_mapping_snapshot** | Build a full snapshot of KYC mapping (`kyc_mapping_snapshot`) with a “most‑recent” window function. |
| **weekly_kyc_fee_load** | Populate audit table `kyc_iccid_wise_country_mapping_audit` with weekly load metadata. |
| **without_dups_inter_loading** | Core “dedup‑and‑enrich” pipeline that reads raw traffic, removes duplicate CDRs, joins KYC, calculates usage metrics, and writes to `traffic_details_inter_raw_daily_with_no_dups`. |
| **mmreport** | Refresh the sponsor master table from a temporary staging table. |
| **nsa5gsub / nsa5gsub_reject** | Load (or reject) 5G NSA product subscription data. |
| **mobilium** | Load Mobilium roaming statistics into `mobilium_mediation_raw_daily`. |
| **pre‑active / pre‑active_reject** | Load pre‑activation SIM records, separating valid rows from rejects (missing mandatory fields). |
| **mnp / mnp_reject** | Load Mobile Number Portability (MNP) details, with a reject path for missing keys. |
| **icusage / icusage_reject** | Load inter‑connect traffic details, with a reject table for null keys. |
| **esimhub / esimhub_reject** | Load API subscription records for eSIM, applying move‑config filters; reject when no matching active SIM. |
| **dpiapplet** | Summarise DPI/Applet service status per month, joining KYC, inventory, and asset‑based service tables. |
| **dm_qosMap / dm_appletProvisioning** | Load QoS metrics and applet provisioning data into VAZ DM tables, partitioned by event date. |
| **lat_long_table_loading** | Load VAZ cell‑level latitude/longitude details. |
| **addonexpiry** | Retrieve active add‑on expiry information for a given `tcl_secs_id` and month. |
| **trafficusages**, **msisdndailyaggr**, **lowbalnotification**, **summonusage** | Parameterised SELECTs used by reporting/alerting jobs (e.g., usage aggregation, low‑balance alerts). |

*All other entries follow the same pattern: a logical name → an INSERT/SELECT statement that moves data between staging, raw, and curated Hive tables.*

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | • Source Hive tables (e.g., `*_inter_raw_daily`, `*_inter_reject_raw_daily`). <br>• External files referenced via `filename` columns (already landed in HDFS). <br>• Runtime parameters supplied by callers: <br> - Date/partition strings (`?` placeholders). <br> - Dynamic DB/table names (`${dbName}`, `${dest_tableName}`, `${src_tableName}`). |
| **Outputs** | • Target Hive tables (mostly partitioned by `partition_date` or `partition_month`). <br>• Audit tables (`*_audit`). <br>• Reject tables for rows failing mandatory‑field checks. |
| **Side‑effects** | • `TRUNCATE` statements (e.g., `kyc_feed_truncate_`) permanently delete data before reload. <br>• `INSERT OVERWRITE` rewrites entire partitions, potentially overwriting existing data. |
| **Assumptions** | • All source tables are refreshed earlier in the pipeline (e.g., raw file ingestion). <br>• Partition columns (`partition_date`, `partition_month`) are derived consistently from filenames or timestamps. <br>• Hive/Impala metastore is up‑to‑date and the `mnaas` database exists. <br>• Environment variables `dbName`, `src_tableName`, `dest_tableName` are defined when the query is rendered. <br>• Caller supplies correct bind values for `?` placeholders (usually a date string). |
| **External Services** | • Hive/Impala execution engine (via `spark‑submit`, `hive -e`, or `impala-shell`). <br>• HDFS for source files (implicitly accessed through Hive external tables). <br>• Optional: `moveconfig` table used by `esimhub*` queries for API whitelist filtering. |

---

### 4. How This File Connects to Other Scripts & Components  

| Consumer Script / Component | How it uses `query.yml` |
|-----------------------------|--------------------------|
| **Bash wrappers** (`traffic_aggr_adhoc_refresh.sh`, `startTestMoveConsumer.sh`, `truncate_table_*.sh`) | Source the file, pick a query name (e.g., `traffic_details_hol`), replace placeholders with environment variables (`${date}`, `${dbName}`), then execute via `hive -e "$SQL"` or `spark-submit`. |
| **Python Spark jobs** (`traffic_nodups_summation.py`) | Load the YAML, retrieve the `without_dups_inter_loading` query, inject Spark parameters (`partition_date`), and run as a Spark SQL job. |
| **Airflow / Oozie DAGs** (not shown but typical) | Reference query names as task parameters; the DAG passes the execution date as the bind value for `?`. |
| **Reporting scripts** (`test.sh`, ad‑hoc reporting) | Use SELECT‑only entries (`trafficusages`, `msisdndailyaggr`, etc.) to fetch data for email or dashboards. |
| **Move‑config service** (`esimhub*`) | Joins with `moveconfig` table to filter API names/statuses; the query is stored here and invoked by the eSIM subscription ingestion pipeline. |
| **Audit / Monitoring** | `weekly_kyc_fee_load` and similar audit queries are called by nightly audit jobs to capture load metadata. |

In practice, the orchestration layer treats the YAML as a *library* of reusable SQL fragments; any new pipeline step only needs to reference the appropriate key.

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss via `TRUNCATE` or `INSERT OVERWRITE`** | Whole partitions can be erased if the job fails after truncation. | • Add a pre‑run backup (e.g., `CREATE TABLE ... AS SELECT * FROM target WHERE partition_date = …`). <br>• Guard truncates with a “dry‑run” flag that logs the intended partitions. |
| **Hard‑coded literals & missing null checks** (e.g., `rejectcode = 0`, `split(filename,'[_]')[0] = 'HOL'`) | Rows that do not meet exact patterns are silently dropped. | • Centralise pattern definitions in a config file. <br>• Add logging of row counts before/after each filter. |
| **Parameter substitution errors** (`?` placeholders, `${dbName}`) | Wrong date or DB name leads to cross‑contamination of data. | • Validate all bound parameters before rendering the SQL (type, format). <br>• Use a templating engine that fails on missing variables. |
| **Schema drift** (source tables gaining new columns) | INSERT statements may fail or silently ignore new fields. | • Keep schema version metadata; run a nightly schema‑compatibility test. |
| **Performance bottlenecks on large joins** (e.g., KYC joins) | Long runtimes, possible OOM in Spark. | • Ensure proper partitioning and bucketing on join keys (`sim`, `iccid`). <br>• Add statistics (`ANALYZE TABLE … COMPUTE STATISTICS`). |
| **Uncontrolled growth of reject tables** | Disk pressure on HDFS. | • Implement a retention policy (e.g., drop rejects older than 30 days). |
| **Missing environment variables** (`dbName`, `src_tableName`) | Query rendering fails, causing pipeline abort. | • Add a startup validation script that checks required env vars. |

---

### 6. Running / Debugging the Queries  

1. **Locate the query** – e.g., `traffic_details_hol`.  
2. **Render the SQL** – most wrappers use a simple `sed`/`envsubst` pattern:  
   ```bash
   export DATE=2024-09-01
   export dbName=mnaas
   sql=$(grep '^traffic_details_hol' config/query.yml | cut -d'>' -f2 | envsubst)
   ```
3. **Execute** – via Hive or Spark:  
   ```bash
   hive -e "$sql"
   # or
   spark-sql -e "$sql"
   ```
4. **Check row counts** – after execution, run a quick count on the target partition:  
   ```sql
   SELECT COUNT(*) FROM mnaas.traffic_details_raw_daily WHERE partition_date='2024-09-01';
   ```
5. **Debug failures** – enable Hive/Impala logging (`set hive.exec.verbose=true;`) or Spark `--conf spark.sql.shuffle.partitions=200`. Capture the rendered SQL in a log file before execution.  
6. **Unit‑test** – create a temporary schema (`mnaas_test`) and run the query against a small synthetic dataset; compare row counts with expected values.

---

### 7. External Config, Environment Variables & File References  

| Variable / Reference | Usage |
|----------------------|-------|
| `${dbName}` | Database name for dynamic table references (e.g., in `siminventory`, `TapErrors`). |
| `${src_tableName}` / `${dest_tableName}` | Used in generic feed loaders to map source and destination tables at runtime. |
| `?` placeholders | Positional bind parameters supplied by the caller (usually a date string or partition key). |
| `moveconfig` table | Provides whitelist/blacklist values for `esimhub*` queries (`con.key='move.api.subscription.list'`). |
| `filename` column | Many queries derive the partition date from the filename using `substr` – assumes a fixed naming convention (`YYYYMMDD…`). |
| `partition_date` / `partition_month` | Partition columns expected by downstream jobs; must be present in source tables. |

---

### 8. Suggested TODO / Improvements  

1. **Migrate to a proper templating engine** (e.g., Jinja2) – this will give clearer error messages for missing variables, support conditional blocks, and avoid ad‑hoc `sed` hacks.  
2. **Introduce a schema‑validation step** before each load – automatically compare source table schemas against the target table definition and abort if mismatched, reducing runtime failures caused by upstream changes.  

--- 

*End of documentation.*