**File:** `mediation-ddls\mnaas_ra_file_rej_geneva.hql`

---

## 1. High‑Level Summary
This script creates the Hive/Impala view `mnaas.ra_file_rej_geneva`. The view consolidates daily traffic details with various reject‑type tables (SEC‑SID, proposition, billing, duplicate) and the “generated data success/reject” table. It produces, per source file and month, the total count of CDRs and the summed usage (minutes for voice, MB for data, count for SMS) broken out by usage type (DAT, VOI, SMS). The view is intended for downstream reporting (e.g., Geneva dashboards) that need a unified view of rejected records across all reject dimensions.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|----------------|
| `ra_file_rej_geneva` | **VIEW** | Exposes aggregated reject metrics per file/month/usage_type. |
| `traffic` (CTE) | **SELECT** | Pulls raw CDR traffic from `traffic_details_raw_daily`, normalises fields (`secs_id`, `proposition`, `usage_type`), and aggregates counts/usage per file/month. |
| `rejects` (CTE) | **UNION** | Combines reject counts from four reject tables (`ba_secsid_reject`, `ba_proposition_reject`, `ba_billing_reject`, `ba_duplicate_reject`) and joins rejected rows from `gen_data_succ_rej` with the traffic CTE to capture detailed reject matches. |
| `mnaas.traffic_details_raw_daily` | **TABLE** | Source of raw CDRs (partitioned by `partition_date`). |
| `mnaas.ba_*_reject` (4 tables) | **TABLES** | Pre‑aggregated reject counts per file/month/usage_type for different reject dimensions. |
| `mnaas.gen_data_succ_rej` | **TABLE** | Holds per‑record success/reject status; used to link traffic rows that were rejected. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.traffic_details_raw_daily` (requires `partition_date`, `filename`, `tcl_secs_id`, `usedtype`, `commercialofferdescription`, `activeaddonname`, `calltype`, `retailduration`, `calldate`).<br>- `mnaas.ba_secsid_reject`, `ba_proposition_reject`, `ba_billing_reject`, `ba_duplicate_reject` (each with columns `filename`, `month`, `usage_type`, `cdr_count`, `cdr_usage`).<br>- `mnaas.gen_data_succ_rej` (columns `status`, `secs_id`, `proposition`, `usage_type`, `month`). |
| **Outputs** | - **VIEW** `mnaas.ra_file_rej_geneva` (columns: `filename`, `month`, `usage_type`, `cdr_count`, `cdr_usage`). |
| **Side Effects** | None beyond DDL (creation/overwrite of the view). |
| **Assumptions** | - Hive/Impala engine with SQL‑92‑compatible syntax.<br>- `partition_date` is stored as a string/date that can be compared to `'2020-09-01'`.<br>- All referenced tables exist and are refreshed before this view is queried.<br>- No column name collisions; `decode` function is available (Hive). |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_ra_file_count_rep_with_reason.hql`** (previous file) | Likely creates the `ba_*_reject` tables that feed this view. |
| **`mnaas_ra_file_geneva_rep.hql`** (previous file) | May consume `ra_file_rej_geneva` for final reporting dashboards. |
| **ETL pipelines that populate `traffic_details_raw_daily`** | Provide the raw CDR data that this view aggregates. |
| **Reject‑generation jobs** (e.g., `mnaas_ra_calldate_*_reject.hql` scripts) | Populate the `ba_*_reject` tables; any change in their schema will affect this view. |
| **Reporting layer (Geneva, PowerBI, etc.)** | Queries the view to display reject metrics per file/month. |
| **Scheduler (e.g., Oozie, Airflow)** | Executes this DDL as part of a nightly “metadata refresh” job after all source tables are refreshed. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Full table scan on `traffic_details_raw_daily`** (no partition pruning beyond `partition_date >= '2020-09-01'`) | High CPU / long query time, especially as data grows. | Add partitioning on `partition_date` (if not already) and ensure the predicate aligns with the partition column. |
| **Data skew on `filename` or `month`** causing long‑running reducers. | Query stalls, OOM errors. | Review distribution; consider bucketing on `filename` or `month` in source tables. |
| **Schema drift in reject tables** (new columns, renamed fields). | View creation fails, downstream reports break. | Implement schema validation step in the pipeline; version control DDL definitions. |
| **Missing or delayed refresh of `gen_data_succ_rej`** leading to under‑reporting of rejects. | Inaccurate metrics. | Add explicit dependency in the scheduler; monitor row‑count delta between source and view. |
| **Incorrect `decode` mapping** (e.g., new `usedtype` values). | Wrong `proposition` values. | Centralise mapping logic in a lookup table or UDF; add unit tests for mapping. |

---

## 6. Running / Debugging the Script  

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_ra_file_rej_geneva.hql
   # or, with Beeline:
   beeline -u jdbc:hive2://<host>:10000/default -f mediation-ddls/mnaas_ra_file_rej_geneva.hql
   ```

2. **Validate Creation**  
   ```sql
   SHOW CREATE VIEW mnaas.ra_file_rej_geneva;
   ```

3. **Sample Query** (quick sanity check)  
   ```sql
   SELECT filename, month, usage_type, cdr_count, cdr_usage
   FROM mnaas.ra_file_rej_geneva
   WHERE month = '2023-07'
   LIMIT 20;
   ```

4. **Debugging Tips**  
   - If the view fails to create, run the inner CTEs individually (`SELECT * FROM traffic LIMIT 10;`, `SELECT * FROM rejects LIMIT 10;`).  
   - Verify that each `ba_*_reject` table contains the expected columns and non‑null values.  
   - Check that `gen_data_succ_rej.status = 'REJECTED'` rows exist for the target month.  
   - Use `EXPLAIN` on the final SELECT to see the execution plan and identify bottlenecks.

---

## 7. External Configuration / Environment Variables  

The script itself does not reference external config files or environment variables. However, typical deployment environments rely on:

| Variable | Purpose |
|----------|---------|
| `HIVE_CONF_DIR` / `BEELINE_OPTS` | Hive/Beeline client configuration (JDBC URL, Kerberos tickets). |
| `DB_HOST`, `DB_PORT` | Connection endpoint for the Hive metastore. |
| `HADOOP_USER_NAME` | Impersonation for file system access (if needed). |

Ensure the execution environment sets these according to the production cluster standards.

---

## 8. Suggested Improvements (TODO)

1. **Materialize the view** – Convert to a scheduled **incremental table** (e.g., using `INSERT OVERWRITE`) to avoid recomputing heavy aggregates on every query, especially as data volume grows.
2. **Parameterise the start date** – Replace the hard‑coded `'2020-09-01'` with a Hive variable (e.g., `${START_DATE}`) so the same script can be reused for back‑fill or test runs without editing the file.  

---