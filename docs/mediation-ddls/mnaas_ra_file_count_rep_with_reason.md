**File:** `mediation-ddls\mnaas_ra_file_count_rep_with_reason.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view **`mnaas.ra_file_count_rep_with_reason`**.  
The view consolidates daily traffic‑detail records (both successful and rejected CDRs) per inbound file, computes aggregate counts and usage metrics (data volume, voice minutes, SMS count), and attaches a rejection‑reason label when a record is flagged as rejected. The result is a single, query‑ready dataset that downstream reporting, reconciliation, and SLA‑monitoring jobs consume to assess file‑level processing health.

---

## 2. Core Logical Components (CTEs) & Their Responsibilities  

| CTE / Section | Purpose | Key Columns Produced |
|---------------|---------|----------------------|
| **`cdr`** | Pulls raw daily traffic details, de‑duplicates records per `(filename, cdrid, chargenumber)` keeping the latest version, and derives per‑record usage flags (data, voice, SMS). | `row_num`, `data_usage`, `sms_usage`, `voi_usage`, `data_count`, `voi_count`, plus all source columns. |
| **`traffic`** | Aggregates the de‑duplicated CDRs to file‑level success metrics: counts of data/voice/SMS CDRs and total usage (MB for data, minutes for voice, count for SMS). | `succ_dat_cdr`, `succ_voi_cdr`, `succ_sms_cdr`, `succ_dat_usg_mb`, `succ_voi_usg_min`, `succ_sms_usg`. |
| **`reject_cdr`** | Extracts rejected raw records, tags each with a *reason* (customer‑number missing, call‑date missing, or `NULL` when none). Also creates per‑record count flags for each call type. | `filename`, `reason`, `data_count`, `sms_count`, `voi_count`. |
| **`reject`** | Summarises rejected CDRs per file and per reason, yielding file‑level reject counts. | `rej_dat_cdr`, `rej_sms_cdr`, `rej_voi_cdr`, `reason`. |
| **`filtered_traffic_file_record_count`** | Provides the baseline file metadata (date, status, total record count) from the ingestion audit table. | `file_date`, `filename`, `file_status`, `record_count`. |
| **Final SELECT** | Left‑joins the three aggregates to the file‑metadata table, coalescing `NULL`s to zero/empty strings, and exposes the full view schema. | `file_date`, `filename`, `file_status`, `file_records`, `success_records`, individual success/reject counts, usage metrics, `reason`. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Source Tables (inputs)** | `mnaas.traffic_details_raw_daily` (raw successful CDRs), `mnaas.traffic_details_raw_reject_daily` (raw rejected CDRs), `mnaas.traffic_file_record_count` (file‑level ingestion audit). |
| **Target Object (output)** | Hive/Impala **VIEW** `mnaas.ra_file_count_rep_with_reason`. The view is *created* (or replaced) each time the script runs; no physical data is written. |
| **Side Effects** | - Overwrites the view definition (DROP/CREATE semantics). <br>- Relies on underlying tables being up‑to‑date; any lag in those tables will be reflected in the view. |
| **Assumptions** | - All referenced tables exist and are refreshed before this script runs (typically by upstream “load” jobs). <br>- Columns used (`filename`, `cdrid`, `chargenumber`, `cdrversion`, `calltype`, `retailduration`, `customernumber`, `calldate`, etc.) are present with expected data types. <br>- Hive/Impala session has `mnaas` database in the search path and sufficient permissions to create views. |
| **External Services** | None directly; the view is consumed by downstream batch jobs, dashboards, or alerting pipelines (e.g., Oozie, Airflow, or custom Java/Python ETL). |

---

## 4. Integration Points with the Rest of the System  

| Consuming Component | How It Connects |
|---------------------|-----------------|
| **Daily reconciliation jobs** (e.g., `mnaas_ra_calldate_*` scripts) | Query this view to compare expected vs. actual record counts per file, feeding SLA dashboards. |
| **Reporting layer** (BI tools, Tableau, PowerBI) | Pulls aggregated metrics (`succ_*`, `rej_*`, usage) for operational dashboards. |
| **Alerting pipelines** (e.g., Spark/Scala jobs that raise alerts on high reject rates) | Filters on `reason <> ''` or `rej_*` thresholds. |
| **Data quality monitoring** (e.g., Deequ or Great Expectations jobs) | Uses the view as a source to validate that `success_records + sum(rej_*) = file_records`. |
| **File‑level audit UI** | Displays `file_status` and `reason` fields for operators to investigate problematic files. |

*Note:* The view name follows the same naming convention as other `mnaas_ra_*` objects, indicating it is part of the “RA” (Reporting/Analytics) data‑model family.

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source‑table schema drift** (e.g., new columns, renamed fields) | View creation fails or returns incorrect aggregates. | Add a pre‑run schema validation step (e.g., `DESCRIBE` checks) and version‑control the script. |
| **Performance degradation** due to the `ROW_NUMBER()` window on large daily tables. | Long view‑creation time, possible OOM errors. | Ensure `traffic_details_raw_daily` is partitioned by `file_date` or `filename`; consider materialising the `cdr` CTE as a temporary table if needed. |
| **Null or malformed `filename` values** causing duplicate rows in joins. | Mis‑aggregated counts, misleading reports. | Enforce NOT NULL constraint on `filename` in upstream ingestion jobs; add a filter `WHERE filename IS NOT NULL` in the view definition. |
| **Incorrect reject reason mapping** (e.g., missing other validation rules). | Operators see empty `reason` while underlying data is actually bad. | Extend the `CASE` in `reject_cdr` to cover additional validation failures as they are identified. |
| **View staleness** if upstream tables are not refreshed before the view is rebuilt. | Reports show outdated numbers. | Schedule this script **after** the daily load jobs (e.g., via Oozie workflow with proper dependencies). |
| **Permission changes** that prevent view creation. | Job failure, downstream pipelines break. | Include a check for `CREATE VIEW` privilege at the start of the script; alert on failure. |

---

## 6. Example Execution & Debugging Workflow  

1. **Run the script** (typical in a CI/CD or scheduler):  
   ```bash
   hive -f mediation-ddls/mnaas_ra_file_count_rep_with_reason.hql
   # or impala-shell -i <impala-host> -f ...
   ```
2. **Verify creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.ra_file_count_rep_with_reason;
   ```
3. **Sample sanity check** (run after view creation):  
   ```sql
   SELECT file_date, filename, file_status,
          success_records, rej_dat_cdr, rej_voi_cdr, reason
   FROM   mnaas.ra_file_count_rep_with_reason
   WHERE  file_date = CURRENT_DATE
   ORDER  BY filename LIMIT 20;
   ```
4. **Debugging tips**  
   - If the view fails to compile, examine the Hive/Impala error log for missing columns.  
   - To isolate performance hotspots, run each CTE individually (e.g., `SELECT * FROM cdr LIMIT 10;`).  
   - Use `EXPLAIN` on the final SELECT to see the query plan and verify that partitions are being pruned.  
   - Compare `file_records` vs. `success_records + rej_*` for a few files to confirm aggregation logic.  

---

## 7. External Configuration / Environment Variables  

| Item | Usage |
|------|-------|
| **Database name (`mnaas`)** | Hard‑coded in the script; can be overridden by setting the default database in the Hive/Impala session (`USE mnaas;`). |
| **Hive/Impala connection parameters** (e.g., `hive.metastore.uris`, `impala.host`) | Provided by the execution environment (Airflow/Oozie job configs, shell scripts). Not referenced directly in the HQL file. |
| **File‑date partitioning** (if upstream tables are partitioned) | The script assumes the daily tables already contain only the relevant date’s data; no explicit partition filter is present. Adjust if partition pruning is required. |

If the deployment uses a templating engine (e.g., Jinja2) to inject schema names, verify that the placeholder substitution occurs before execution.

---

## 8. Suggested TODO / Improvements  

1. **Add explicit partition pruning** – modify the `traffic_details_raw_daily` and `traffic_details_raw_reject_daily` references to include `WHERE file_date = ${process_date}` (or similar) so the view only scans the current day’s partition, dramatically reducing runtime.  
2. **Materialise the view as a daily table** – replace the view with a **partitioned table** (`ra_file_count_rep_with_reason`) that is refreshed each night. This will give downstream jobs faster reads and allow incremental updates if needed.  

---