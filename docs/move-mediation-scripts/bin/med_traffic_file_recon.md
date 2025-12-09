# Summary
`med_traffic_file_recon.hql` is a Hive‑SQL script executed via the Hive CLI or Beeline in a Spark execution engine. It processes daily raw traffic files (accepted and rejected) for a given `filedate`, aggregates usage metrics per file and per `tcl_secs_id`, determines Geneva automation status, and writes the results into the partitioned table `mnaas.med_traffic_file_recon` (both overwrite for accepted data and insert for rejected data). The script is part of the Move‑Mediation nightly pipeline that reconciles traffic file ingest with downstream billing/analytics.

# Key Components
- **Hive session settings** – forces Spark engine, enables dynamic partitions, sets non‑strict mode.  
- **INSERT OVERWRITE** – populates `mnaas.med_traffic_file_recon` partition `file_date` with aggregated *accepted* traffic (`traffic_details_raw_daily`).  
- **INSERT INTO** – appends aggregated *rejected* traffic (`traffic_details_raw_reject_daily`) to the same partition.  
- **Derived columns**  
  - `Med_File_Type` – `substring(filename,1,3)`.  
  - Normalized `calltype` → `DAT/VOI/SMS`.  
  - `Geneva_Automation` – business rule based on file prefix and product code presence.  
  - `Load_Status` – literal `'Accepted'` or `'Rejected'`.  
- **Aggregations** – `count(1)`, `count(distinct cdrid)`, `sum(retailduration)`.  
- **Join** – left outer join with `mnaas.gen_usage_product` to resolve product code and termination date.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1 | Hive variables: `${hiveconf:filedate}` (YYYYMMDD) | Filter `traffic_details_raw_daily` / `traffic_details_raw_reject_daily` on `substring(filename,35,6) = filedate` | Aggregated rows per file & SECS ID | Overwrites partition `file_date` for accepted data; inserts new rows for rejected data |
| 2 | `mnaas.gen_usage_product` (product metadata) | Left outer join on `tcl_secs_id` and normalized calltype | Determines `Geneva_Automation` flag and filters out terminated plans (`prd.plan_termination_date is null` for rejects) | None |
| 3 | Aggregation functions | Compute CDR counts, distinct CDR counts, total retail duration | Rows written to `mnaas.med_traffic_file_recon` | Table partition updated; downstream reporting consumes this table |

External services: Hive metastore (for table definitions), HDFS (source raw tables and target table storage), Spark execution engine.

# Integrations
- **Upstream**: Daily ingestion jobs that populate `mnaas.traffic_details_raw_daily` and `mnaas.traffic_details_raw_reject_daily` from FTP/SFTP traffic files.  
- **Downstream**: Reporting components (`ReportWriter`, `TextReader`) that query `mnaas.med_traffic_file_recon` for reconciliation dashboards and SLA monitoring.  
- **Orchestration**: Typically invoked by an Oozie or Airflow DAG that sets `filedate` and triggers the script via `hive -f med_traffic_file_recon.hql -hiveconf filedate=20251207`.  

# Operational Risks
- **Incorrect `filedate`** → No rows processed; partition remains empty. *Mitigation*: Validate variable presence and format before execution.  
- **Schema drift in source tables** (e.g., new columns, renamed fields) → Query failures. *Mitigation*: Version control schema changes and include defensive `SELECT` list.  
- **Join explosion** if `gen_usage_product` contains many duplicate `secs_code`/`product_code` combos → performance degradation. *Mitigation*: Ensure unique constraints or pre‑aggregate product metadata.  
- **Dynamic partition misconfiguration** → data written to wrong partition or Hive metastore inconsistency. *Mitigation*: Keep `hive.exec.dynamic.partition.mode=nonstrict` and monitor Hive logs.  
- **Spark resource contention** during peak windows → job timeouts. *Mitigation*: Allocate dedicated YARN queue and set appropriate `spark.executor.memory`.  

# Usage
```bash
# Example execution for 2025‑12‑07
export HIVE_CONF_DIR=/etc/hive/conf
hive -f /opt/move-mediation-scripts/bin/med_traffic_file_recon.hql \
     -hiveconf filedate=20251207
```
To debug, prepend `set -v;` in the script or run with `hive -e "set hive.exec.verbose=true; ..."`. Check `hive.log` for partition creation messages.

# Configuration
- **Hive variables**: `filedate` (mandatory, format `yyyyMMdd`).  
- **Hive session properties** (hard‑coded in script):  
  - `hive.execution.engine=spark`  
  - `hive.exec.dynamic.partition=true`  
  - `hive.exec.dynamic.partition.mode=nonstrict`  
- **External tables**: `mnaas.traffic_details_raw_daily`, `mnaas.traffic_details_raw_reject_daily`, `mnaas.gen_usage_product`, `mnaas.med_traffic_file_recon`.  
- No external config files referenced; all logic is self‑contained.

# Improvements
1. **Parameter validation block** – add a pre‑execution check that aborts if `${hiveconf:filedate}` is missing or does not match `^[0-9]{8}$`.  
2. **Modularize business rules** – extract `Geneva_Automation` logic into a UDF or a lookup table to simplify maintenance when product‑code rules evolve.