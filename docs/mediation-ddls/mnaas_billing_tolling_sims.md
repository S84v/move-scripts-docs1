**File:** `mediation-ddls\mnaas_billing_tolling_sims.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates the view `mnaas.billing_tolling_sims`. The view consolidates “tolling” SIM records from two sources – current daily traffic details and late‑arriving CDRs – and flags each SIM as billable (`Yes`/`No`). It is scoped to records whose source file name contains “HOL” (holiday‑related traffic) and is used downstream by billing‑related jobs (e.g., `mnaas_billing_tolling_eids.hql`) to determine which SIMs should be included in toll‑based charge calculations.

---

## 2. Key Objects & Responsibilities

| Object | Type | Responsibility |
|--------|------|-----------------|
| `curr_toll_sims` | CTE | Pulls distinct SIMs from `traffic_details_raw_daily` (joined implicitly with `month_billing`) for the current billing window, marks every row as billable (`'Yes'`). |
| `late_toll_sims` | CTE | Pulls distinct SIMs from late‑arriving CDRs (`traffic_details_billing_late_cdr`) and matches them against the master list `tolling_sim_list`. Determines billable flag: `'Yes'` if the SIM appears in the master list, otherwise `'No'`. |
| `all_toll_sims` | CTE | Union of the two CTEs, providing a complete set of tolling SIMs for the period. |
| `billing_tolling_sims` | View | Exposes `callmonth`, `billable`, `tcl_secs_id`, `commercialoffer`, `sim` for all tolling SIMs where a commercial offer description exists. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input tables** | `mnaas.traffic_details_raw_daily`, `mnaas.month_billing`, `mnaas.tolling_sim_list`, `mnaas.traffic_details_billing_late_cdr` |
| **Key columns used** | `partition_date`, `start_date`, `end_date` (from `month_billing`), `filename`, `tcl_secs_id`, `commercialofferdescription`, `sim`, `calldate`, `bill_month` |
| **Filters** | `partition_date BETWEEN start_date AND end_date` <br> `filename LIKE '%HOL%'` <br> `tcl_secs_id > 0` <br> `commercialofferdescription != ''` |
| **Output** | Hive view `mnaas.billing_tolling_sims` (no physical table is written). |
| **Side effects** | Creation/overwrite of the view. No data mutation. |
| **External dependencies** | Hive/Impala execution engine, underlying HDFS storage, any Kerberos or LDAP authentication configured for the `mnaas` database. |
| **Assumptions** | - All referenced tables exist and have the columns used here.<br>- `month_billing` contains a single row for the current billing period (provides `start_date`, `end_date`, `month`).<br>- `filename` field contains the string “HOL” for holiday traffic only.<br>- The environment supplies appropriate Hive/Impala connection parameters (usually via a wrapper script or Airflow operator). |

---

## 4. Integration Points

| Connected Component | How it connects |
|---------------------|-----------------|
| **`mnaas_billing_tolling_eids.hql`** | Likely consumes `billing_tolling_sims` to join with EID (equipment ID) tables for final charge aggregation. |
| **`traffic_details_raw_daily` & `traffic_details_billing_late_cdr` ingestion jobs** | Populate the raw and late‑CDR tables that feed this view. |
| **`month_billing` generation script** | Supplies the billing window (`start_date`, `end_date`, `month`) used in the view’s date filter. |
| **`tolling_sim_list` maintenance job** | Updates the master list of SIMs that are considered billable for tolling; changes affect the `billable` flag in this view. |
| **Downstream billing pipelines (e.g., nightly aggregation, reporting)** | Query `billing_tolling_sims` to decide which SIMs generate toll‑based revenue. |
| **Orchestration layer (Airflow, Oozie, etc.)** | Executes this DDL as part of a “create‑views” task before downstream ETL jobs run. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – underlying tables change (column rename, type change). | View creation fails or returns wrong data. | Add automated schema‑validation step before view creation; version‑control DDL changes. |
| **Performance degradation** – full outer join on large late‑CDR table. | Long view creation time, downstream jobs delayed. | Ensure `traffic_details_billing_late_cdr` is partitioned by `bill_month`; add appropriate statistics; consider rewriting as left join if business logic permits. |
| **Incorrect billable flag** – missing entries in `tolling_sim_list`. | Revenue leakage (unbilled usage). | Periodic reconciliation report comparing `billable='Yes'` vs master list; alert on sudden drop in count. |
| **Stale view** – view not refreshed after source data reload. | Downstream jobs use outdated SIM set. | Include view recreation in the same transaction as source table loads or schedule a “refresh‑view” task after data load. |
| **File‑name filter mismatch** – change in naming convention (e.g., “HOL” becomes “HOLIDAY”). | Relevant records omitted. | Externalize the pattern (`%HOL%`) to a config property; add unit test that verifies at least one row is returned for a known sample file. |

---

## 6. Running / Debugging Guide

1. **Execute the script**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_tolling_sims.hql
   # or via beeline:
   beeline -u jdbc:hive2://<host>:10000/mnaas -f mediation-ddls/mnaas_billing_tolling_sims.hql
   ```

2. **Validate view creation**  
   ```sql
   SHOW CREATE VIEW mnaas.billing_tolling_sims;
   ```

3. **Inspect intermediate CTE results** (use temporary tables or `INSERT OVERWRITE` to a staging table for debugging):  
   ```sql
   CREATE TEMPORARY TABLE tmp_curr AS
   SELECT * FROM (
       SELECT DISTINCT tcl_secs_id, commercialofferdescription commercialoffer,
              sim, substring(calldate,1,7) callmonth, 'Yes' billable
       FROM mnaas.traffic_details_raw_daily
       JOIN mnaas.month_billing
         ON partition_date BETWEEN start_date AND end_date
       WHERE filename LIKE '%HOL%' AND tcl_secs_id > 0
         AND commercialofferdescription <> ''
   ) t;
   SELECT COUNT(*) FROM tmp_curr;
   ```

4. **Check row counts** to ensure expected volume:  
   ```sql
   SELECT callmonth, billable, COUNT(*) AS cnt
   FROM mnaas.billing_tolling_sims
   GROUP BY callmonth, billable;
   ```

5. **Log & monitoring**  
   - Capture Hive query logs (stderr/stdout) in the orchestration system.  
   - Set up a metric on the row count of the view; alert if count deviates >20% from historical baseline.

---

## 7. External Configuration / Environment Variables

| Config Item | Usage |
|-------------|-------|
| Hive/Impala connection URL, user, password | Provided by the orchestration wrapper (e.g., Airflow `HiveOperator`). |
| `HIVE_CONF_DIR` / Kerberos ticket cache | Required for secure clusters; not referenced directly in the script but must be present in the runtime environment. |
| Optional: `HOLIDAY_FILE_PATTERN` (not currently used) | Could be introduced to replace the hard‑coded `'%HOL%'` filter. |

---

## 8. Suggested Improvements (TODO)

1. **Parameterize the filename filter** – replace the hard‑coded `'%HOL%'` with a configurable variable (e.g., `${HOLIDAY_PATTERN}`) so the view can be reused for other traffic categories without code change.  
2. **Add explicit partition pruning** – ensure `traffic_details_raw_daily` and `traffic_details_billing_late_cdr` are partitioned on `partition_date` / `bill_month` and include `WHERE` clauses that reference those partitions directly to improve query performance.  

---