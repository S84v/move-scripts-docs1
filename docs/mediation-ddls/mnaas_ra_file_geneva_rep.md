**File:** `mediation-ddls\mnaas_ra_file_geneva_rep.hql`  

---

## 1. High‑Level Summary
This script creates the Hive/Impala view **`mnaas.ra_file_geneva_rep`**. The view aggregates per‑file, per‑month CDR (Call Detail Record) statistics for both successfully processed files (from `gen_file_cdr_mapping`) and rejected records (from `ra_calldate_rej_tbl`). It normalises the different call‑type columns (Data, SMS, A2P, Voice) into unified “generated” and “rejected” metrics, concatenates source file names, and presents a single row per file/month that downstream reporting jobs can consume (e.g., Geneva‑style revenue/usage reports).

---

## 2. Important Logical Blocks (CTEs) & Their Responsibilities  

| Block | Purpose | Key Columns Produced |
|-------|---------|----------------------|
| **`success_files`** | Filters `gen_file_cdr_mapping` for rows where `mapping_status = 'SUCCESS'` and pivots the `calltype` values into separate generated‑CDR and usage columns. | `filename`, `gen_dat_cdr`, `gen_dat_usage`, `gen_sms_cdr`, `gen_sms_usage`, `gen_a2p_cdr`, `gen_a2p_usage`, `gen_voi_cdr`, `gen_voi_usage`, `gen_file_name`, `month` |
| **`gen_success`** | Summarises `success_files` per `filename`/`month`, aggregates counts/usage, merges SMS & A2P into a single SMS metric, and concatenates distinct source file names. | `filename`, `gen_dat_cdr`, `gen_dat_usage`, `gen_sms_cdr`, `gen_sms_usage`, `gen_voi_cdr`, `gen_voi_usage`, `gen_file_name`, `month` |
| **`rej_files`** | Reads raw reject records from `ra_calldate_rej_tbl` and pivots `usage_type` (DAT, SMS, A2P, VOI) into separate reject‑CDR and usage columns. | `filename`, `rej_dat_cdr`, `rej_dat_usage`, `rej_sms_cdr`, `rej_sms_usage`, `rej_a2p_cdr`, `rej_a2p_usage`, `rej_voi_cdr`, `rej_voi_usage`, `callmonth` |
| **`gen_reject`** | Summarises `rej_files` per `filename`/`callmonth`, merges SMS & A2P reject counts/usage, and produces a per‑file reject metric set. | `filename`, `rej_dat_cdr`, `rej_dat_usage`, `rej_sms_cdr`, `rej_sms_usage`, `rej_voi_cdr`, `rej_voi_usage`, `callmonth` |
| **Final SELECT** | Performs a **FULL OUTER JOIN** between `gen_success` and `gen_reject` on `filename` and month, coalescing nulls, and outputs the unified view columns. | `filename`, `month`, all generated & rejected CDR/usage columns, `gen_file_name` |

---

## 3. Inputs, Outputs & Side Effects  

| Aspect | Details |
|--------|---------|
| **Source Tables** | `mnaas.gen_file_cdr_mapping` (contains per‑file CDR generation metadata) <br> `mnaas.ra_calldate_rej_tbl` (contains rejected call‑date records) |
| **Target Object** | Hive/Impala **VIEW** `mnaas.ra_file_geneva_rep` (read‑only logical table) |
| **Side Effects** | DDL – creates/overwrites the view. No data mutation. |
| **Assumptions** | • `mapping_status` values are limited to `'SUCCESS'` for good rows.<br>• `calltype` and `usage_type` contain only the enumerated values (Data, SMS, A2P, Voice / DAT, SMS, A2P, VOI).<br>• `month` and `callmonth` are comparable (same datatype, same calendar month granularity). |
| **External Dependencies** | • Hive/Impala execution engine (configured via environment variables such as `HIVE_CONF_DIR`, `IMPALA_HOST`).<br>• Underlying tables must be present and up‑to‑date (created by earlier DDL scripts in the mediation‑ddls folder). |

---

## 4. Integration with the Rest of the Mediation System  

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| `mnaas_ra_calldate_rej_tbl` DDL (e.g., `mnaas_ra_calldate_rej_rep.hql`) | Provides the reject table referenced in `rej_files`. |
| `mnaas_gen_file_cdr_mapping` DDL (e.g., `mnaas_ra_file_count_rep_with_reason.hql`) | Supplies the success mapping table used in `success_files`. |
| Down‑stream reporting jobs (e.g., Geneva revenue extraction, BI dashboards) | Query `ra_file_geneva_rep` to obtain per‑file, per‑month usage & reject metrics. |
| Data quality / audit scripts | May compare aggregates from this view against raw source tables to validate ETL completeness. |
| Partition‑maintenance scripts (if the view is later materialised) | Could refresh a materialised version of the view on a nightly schedule. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Full outer join on large tables** may cause high memory/CPU usage and long run times. | Job stalls, cluster resource exhaustion. | • Ensure both source tables are **partitioned** on `filename`/`month`.<br>• Gather statistics (`ANALYZE TABLE … COMPUTE STATISTICS`) before view creation.<br>• Consider materialising the view nightly if latency is acceptable. |
| **Null handling** – coalescing columns may hide missing data. | Inaccurate reporting (e.g., missing reject counts). | • Add explicit `CASE` statements or default values in the final SELECT.<br>• Include a QA step that validates row counts against source tables. |
| **Schema drift** – new `calltype` or `usage_type` values break the `DECODE` logic. | Silent loss of new service types. | • Centralise the mapping logic in a lookup table and join instead of hard‑coded `DECODE`.<br>• Add unit tests that fail when unexpected values appear. |
| **Dependency order** – view creation before source tables exist leads to compilation errors. | Deployment failures. | • Enforce a **dependency graph** in the CI/CD pipeline (e.g., using Airflow DAG or custom script). |
| **Permissions** – view may be created with insufficient privileges for downstream users. | Access denied errors. | • Grant `SELECT` on the view to the appropriate roles/groups after creation. |

---

## 6. Running / Debugging the Script  

1. **Typical Execution** (as part of the ETL batch):  
   ```bash
   hive -f mediation-ddls/mnaas_ra_file_geneva_rep.hql
   # or, for Impala:
   impala-shell -f mediation-ddls/mnaas_ra_file_geneva_rep.hql
   ```

2. **Verification Steps**  
   *Check view definition*:  
   ```sql
   SHOW CREATE VIEW mnaas.ra_file_geneva_rep;
   ```
   *Sample data*:  
   ```sql
   SELECT * FROM mnaas.ra_file_geneva_rep LIMIT 20;
   ```
   *Row‑count sanity*:  
   ```sql
   SELECT COUNT(*) FROM mnaas.ra_file_geneva_rep;
   SELECT COUNT(*) FROM mnaas.gen_file_cdr_mapping WHERE mapping_status='SUCCESS';
   SELECT COUNT(*) FROM mnaas.ra_calldate_rej_tbl;
   ```

3. **Debugging Tips**  
   * Isolate each CTE: replace the final SELECT with `SELECT * FROM success_files LIMIT 10;` and repeat for other CTEs.  
   * Verify that `month` and `callmonth` are comparable (same datatype).  
   * If the view fails to compile, check Hive/Impala logs for “cannot resolve column” – often a typo in column names or missing table.  

---

## 7. Configuration / External References  

| Item | Usage |
|------|-------|
| **Database name** (`mnaas`) | Hard‑coded in the script; must exist in the Hive metastore. |
| **Environment variables** (e.g., `HIVE_CONF_DIR`, `IMPALA_HOST`) | Not referenced directly in the script but required for the Hive/Impala client that runs the `.hql`. |
| **Table/column definitions** | Defined in other DDL files within `mediation-ddls`; ensure they are applied before this script runs. |
| **Cluster resources** (YARN queues, Impala memory limits) | Should be sized to accommodate the full outer join; may be tuned via cluster configuration. |

---

## 8. Suggested TODO / Improvements  

1. **Replace hard‑coded `DECODE` logic with a lookup table** – create a small reference table (`calltype_mapping`) that maps `calltype`/`usage_type` to logical columns (e.g., `gen_dat_cdr`). Join this table in the CTEs to make the view resilient to new service types.  

2. **Materialise the view as a partitioned table** (e.g., `ra_file_geneva_rep_mv` partitioned by `month`) if downstream consumption is frequent. This will dramatically improve query performance and allow incremental refreshes.  

--- 

*End of documentation.*