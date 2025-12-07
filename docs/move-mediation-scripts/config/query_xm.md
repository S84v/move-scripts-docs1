**File:** `move-mediation-scripts/config/query_xm.yml`  

---

## 1. One‑sentence production summary
A central YAML‑style mapping that stores all Hive/Impala DML statements used by the “move‑mediation” pipeline – from raw CDR ingestion through de‑duplication, enrichment, aggregation, and reject handling – allowing the Bash/Python orchestration layer to retrieve the appropriate SQL by logical name and execute it against the MNAAS data‑warehouse.

---

## 2. Core “objects” (keys) and their responsibilities  

| Logical name (key) | Primary purpose | Target table(s) | Typical execution stage |
|--------------------|----------------|-----------------|------------------------|
| `traffic_details_hol` / `traffic_details_sng` | Load raw CDRs (HOL‑type or SNG‑type) into `traffic_details_raw_daily` (partitioned) | `traffic_details_inter_reject_raw_daily_2023` → `traffic_details_raw_daily` | **Ingestion / Initial load** |
| `siriusxm_reject_query` | Persist rejected SiriusXM CDRs for later re‑processing | `traffic_details_inter_raw_daily_2023` → `siriusxm_reject_data` | **Reject capture** |
| `siriusxm_reProcessingQuery` | Re‑process previously rejected SiriusXM records after KYC enrichment | `kyc_snapshot` + `siriusxm_reject_data` → `traffic_details_raw_daily` | **Reject re‑process** |
| `activations_hol` / `activations_sng` | Load activation records (HOL / SNG) into `activations_raw_daily` | `activations_inter_raw_daily` → `activations_raw_daily` | **Activation ingest** |
| `actives_hol` / `actives_sng` | Load active‑SIM records (HOL / SNG) into `actives_raw_daily` | `actives_inter_raw_daily` → `actives_raw_daily` | **Active‑SIM ingest** |
| `tolling_hol` / `tolling_sng` | Generic “tolling” loader – table name placeholders (`${dbName}`, `${src_tableName}`, `${dest_tableName}`) | Variable | **Generic loader** |
| `siminventory`, `TapErrors`, `FailedEvents`, `IMEIChange_feed`, `IPVProbe_feed` | Load various auxiliary feeds (SIM inventory, TAP errors, failed events, IMEI changes, IPV probe) into partitioned tables | Variable | **Feed ingestion** |
| `kyc_feed_truncate_` | Truncate staging table before a KYC load | `kyc_mapping_snapshot` | **Pre‑load housekeeping** |
| `kyc_feed_load` | Load raw KYC dump into `kyc_iccid_wise_country_mapping` | `kyc_inter_raw_daily` → `kyc_iccid_wise_country_mapping` | **KYC raw load** |
| `kyc_feed_snapshot` | Build the current KYC snapshot (`kyc_snapshot`) from the mapping table | `kyc_iccid_wise_country_mapping` → `kyc_snapshot` | **KYC snapshot** |
| `kyc_feed_mapping_snapshot` | Build a historical KYC mapping snapshot (`kyc_mapping_snapshot`) | `kyc_iccid_wise_country_mapping` → `kyc_mapping_snapshot` | **KYC mapping snapshot** |
| `weekly_kyc_fee_load` | Load weekly KYC audit rows | `kyc_inter_raw_weekly` → `kyc_iccid_wise_country_mapping_audit` | **Audit** |
| `addonexpiry`, `trafficusages`, `msisdndailyaggr`, `lowbalnotification`, `summonusage` | Parameterised SELECTs used by downstream reporting / alerting jobs (often invoked via Spark or Hive) | Various `traffic_*` tables | **Reporting / analytics** |
| `lat_long_table_loading` | Load VAZ (cell‑site) details into `vaz_details_raw_daily` | `vaz_details_inter_raw_daily` → `vaz_details_raw_daily` | **Geolocation enrichment** |
| `without_dups_inter_loading` | De‑duplicate raw traffic, enrich with KYC, compute usage metrics, write to `traffic_details_inter_raw_daily_2023_with_no_dups` | `traffic_details_raw_daily` + `kyc_mapping_snapshot` → `traffic_details_inter_raw_daily_2023_with_no_dups` | **Deduplication & enrichment** |
| `mmreport` | Refresh the MMR sponsor master table from a temp staging table | `mmr_sponsor_master_temp` → `mmr_sponsor_master` | **Master data refresh** |
| `nsa5gsub`, `nsa5gsub_reject` | Load (or reject) 5G NSA subscription data | `nsa_fiveg_product_subscription_temp` → `nsa_fiveg_product_subscription` / reject table | **5G subscription handling** |
| `mobilium` | Load Mobilium roaming statistics | `mobilium_mediation_raw_daily_temp` → `mobilium_mediation_raw_daily` | **Roaming stats** |
| `pre-active`, `pre-active_reject` | Load pre‑activation SIM data (valid / reject) | `traffic_pre_active_raw_sim_temp` → `traffic_pre_active_raw_sim` / reject | **Pre‑activation ingest** |
| `mnp`, `mnp_reject` | Load Mobile Number Portability (MNP) details (valid / reject) | `mnp_portinout_details_temp` → `mnp_portinout_details_raw` / reject | **MNP ingest** |
| `icusage`, `icusage_reject` | Load inter‑connect traffic usage (valid / reject) | `interconnect_traffic_details_temp` → `interconnect_traffic_details_raw` / reject | **Inter‑connect usage** |
| `esimhub`, `esimhub_reject` | Load eSIM API subscription events (valid / reject) with active‑SIM join and config filter (`moveconfig`) | `api_subscription_temp` → `api_subscription_raw` / reject | **eSIM subscription handling** |

*Note:* Keys that contain placeholders (`${dbName}`, `${src_tableName}`, `${dest_tableName}`) are interpolated by the calling script (usually a Bash wrapper) before execution.

---

## 3. Inputs, outputs & side‑effects  

| Category | Details |
|----------|---------|
| **Inputs** | • Source Hive/Impala tables (e.g., `traffic_details_inter_reject_raw_daily_2023`, `kyc_snapshot`, `api_subscription_temp`). <br>• Runtime parameters supplied by the orchestrator (e.g., `partition_date`, `partition_month`, `?` placeholders, `${dbName}` etc.). |
| **Outputs** | • Inserts/overwrites into target Hive tables (partitioned by `partition_date` or `partition_month`). <br>• Occasionally a `TRUNCATE TABLE` (KYC feed). |
| **Side‑effects** | • Table partitions are created/overwritten, potentially causing downstream jobs to see new data. <br>• `TRUNCATE` removes all rows from a staging table, so any concurrent load must be coordinated. |
| **Assumptions** | • Hive/Impala metastore is reachable and the target database (`mnaas`) exists. <br>• All referenced tables have the expected schema (column order matches the SELECT list). <br>• Partition columns (`partition_date`, `partition_month`) are strings in `yyyy‑MM‑dd` / `yyyy‑MM` format. <br>• The calling script performs proper escaping of single quotes and substitutes placeholders safely. |

---

## 4. How this file connects to other components  

1. **Orchestration wrappers** – Bash scripts in `move-mediation-scripts/bin/` (e.g., `traffic_nodups_summation.sh`, `usage_overage_charge.sh`) read this YAML via `yq`/`awk` or a small Python helper to fetch the SQL for a given logical name, replace placeholders (`?`, `${dbName}` etc.), and feed the final statement to `hive -e` or `impala-shell -q`.  

2. **Spark jobs** – Some pipelines (e.g., `traffic_nodups_summation.py`) load the same queries via a Python YAML parser to build DataFrames with `spark.sql()`.  

3. **Configuration layer** – `moveconfig` table (referenced in `esimhub` queries) stores runtime filters (allowed API names / statuses). The query joins this table to enforce business rules.  

4. **Dependency chain** –  
   * `traffic_details_hol` / `traffic_details_sng` → `traffic_details_raw_daily` → `traffic_nodups_summation.py` (aggregation).  
   * `kyc_feed_*` → `kyc_snapshot` → many enrichment joins (e.g., `siriusxm_reProcessingQuery`).  
   * `without_dups_inter_loading` → `traffic_details_inter_raw_daily_2023_with_no_dups` → downstream usage/aggregation queries (`trafficusages`, `summonusage`).  

5. **External services** – No direct network calls; all operations are internal to the Hive/Impala cluster. However, downstream jobs may export data to S3, FTP, or downstream billing systems.

---

## 5. Operational risks & mitigations  

| Risk | Impact | Recommended mitigation |
|------|--------|------------------------|
| **Schema drift** – source tables change (column added/removed) while the SELECT list stays static. | Insert failures, silent data loss, downstream job breakage. | Implement automated schema‑validation step (e.g., `DESCRIBE` + compare to expected column list) before running each query. |
| **Partition mis‑alignment** – wrong `partition_date` value leads to data landing in the wrong partition or overwriting existing data. | Incorrect reporting, data loss. | Enforce strict date‑format validation in the wrapper; log the actual partition value used. |
| **Unbounded `TRUNCATE`** – `kyc_feed_truncate_` runs while another process is still reading the table. | Empty reads, job failures. | Serialize KYC load steps via a lock file or Airflow semaphore; add a “dry‑run” flag that skips truncate. |
| **Placeholder injection** – malformed `${dbName}` or `${src_tableName}` values cause SQL syntax errors or cross‑database writes. | Job abort, possible data leakage. | Whitelist allowed values; perform sanity checks (regex) before substitution. |
| **Large‑scale inserts** – some queries insert millions of rows without `INSERT … SELECT … LIMIT`. | Hive/Impala OOM, long runtimes. | Tune Hive/Impala execution parameters (e.g., `set hive.exec.reducers.bytes.per.reducer=256000000;`) and monitor job duration. |
| **Missing join keys** – e.g., `esimhub` joins on `moveconfig` with `array_contains` – if config is missing, rows go to reject table silently. | Undetected data quality issues. | Add a post‑run validation that reject count < threshold; alert if exceeded. |

---

## 6. Typical operator / developer workflow  

1. **Locate the logical query**  
   ```bash
   # Example: need the HOL traffic load
   QUERY=$(yq e '.traffic_details_hol' move-mediation-scripts/config/query_xm.yml)
   ```

2. **Inject runtime parameters** (date, db name, etc.)  
   ```bash
   PARTITION_DATE=$(date -d 'yesterday' +%Y-%m-%d)
   DB=mnaas
   SRC_TABLE=traffic_details_inter_reject_raw_daily_2023
   DEST_TABLE=traffic_details_raw_daily

   # Replace placeholders (if any)
   FINAL_SQL=$(echo "$QUERY" | sed -e "s/\${dbName}/$DB/g" \
                                 -e "s/\${src_tableName}/$SRC_TABLE/g" \
                                 -e "s/\${dest_tableName}/$DEST_TABLE/g")
   ```

3. **Execute** (via Hive or Impala)  
   ```bash
   hive -e "$FINAL_SQL"
   # or
   impala-shell -q "$FINAL_SQL"
   ```

4. **Validate**  
   * Check Hive metastore for new partition: `SHOW PARTITIONS mnaas.traffic_details_raw_daily;`  
   * Run a quick count: `SELECT COUNT(*) FROM mnaas.traffic_details_raw_daily WHERE partition_date='$PARTITION_DATE';`  

5. **Debugging**  
   * If the query fails, capture the full error log (`$HIVE_ERR_LOG`).  
   * Verify that all source tables exist and are populated for the target date.  
   * Run the SELECT part alone (without `INSERT`) to inspect column alignment.  

6. **Automation**  
   * In Airflow/DAG or cron, the same steps are wrapped in a Python operator that loads the YAML, renders the SQL with Jinja2, and calls `HiveOperator` / `ImpalaOperator`.  

---

## 7. External configuration / environment variables referenced  

| Variable / external file | Usage |
|--------------------------|-------|
| `${dbName}`, `${src_tableName}`, `${dest_tableName}` | Runtime placeholders interpolated by the calling script (usually Bash). |
| `moveconfig` table (`moveconfig.key='move.api.subscription.list'`) | Used in `esimhub*` queries to filter allowed API names and statuses. |
| `?` placeholders in some queries (e.g., `trafficusages`, `msisdndailyaggr`) | Positional parameters supplied by the orchestrator (often via `hive -hiveconf` or Spark `sqlContext.sql(query, param1, param2)`). |
| `CURRENT_TIMESTAMP`, `date_format(current_date,'yyyy-MM')` | Hive built‑ins – no external config needed. |
| `yq`/`python yaml` parsers | Not part of this file but required by scripts that read it. |

---

## 8. Suggested improvements (TODO)

1. **Schema‑driven templating** – Replace the long hand‑crafted column lists with a generated list based on the target table schema (e.g., using Hive `SHOW CREATE TABLE`). This reduces maintenance when columns are added/removed.

2. **Centralised placeholder validation** – Introduce a small validation script (`validate_query_xm.yml`) that scans the file for `${...}` tokens and ensures every token has a corresponding environment variable or default value, failing fast before any job runs.  

--- 

*End of documentation.*