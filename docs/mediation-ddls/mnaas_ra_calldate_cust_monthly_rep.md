**File:** `mediation-ddls\mnaas_ra_calldate_cust_monthly_rep.hql`

---

### 1. Summary
This script creates the Hive view **`mnaas.ra_calldate_cust_monthly_rep`**. The view aggregates, per billing month and organization (TCL/SEC ID), the *mediation*, *rejection*, and *generation* counts and usage volumes for four usage types (DAT, SMS, A2P, VOI). It joins the organization master (`org_details`) to expose the readable organization name. The view is intended for downstream monthly reporting, billing, and analytics pipelines.

---

### 2. Core Logical Blocks (CTEs) & Their Responsibilities  

| CTE / Component | Purpose |
|-----------------|---------|
| **`med_data`** | Pulls raw mediation rows from `ra_calldate_traffic_table` and pivots them into separate columns for each usage type (CDR count & usage). |
| **`med_summary`** | Summarises `med_data` by `callmonth` & `tcl_secs_id`, aggregating counts and converting raw usage to MB (data/VOI) or raw units. |
| **`rej_data`** | Extracts rejected call records from `ra_calldate_rej_tbl` (filtered by the current `month` from `month_reports`) and pivots them similarly. |
| **`rej_summary`** | Aggregates `rej_data` by month & organization. |
| **`gen_data`** | Retrieves successful generation records from `ra_calldate_gen_succ_rej` (filtered by `status='SUCCESS'` and the current `month`) and pivots them. |
| **`gen_summary`** | Aggregates `gen_data` by month & organization. |
| **Final SELECT** | Joins the three summaries (med, rej, gen) together and adds the organization name from `org_details`. Produces the final column set for the view. |

---

### 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Input tables** | `mnaas.ra_calldate_traffic_table`<br>`mnaas.ra_calldate_rej_tbl`<br>`mnaas.ra_calldate_gen_succ_rej`<br>`mnaas.month_reports`<br>`mnaas.org_details` |
| **External parameter** | Hive variable **`month`** (used to filter `month_reports.bill_month`). Must be supplied at execution time. |
| **Output** | Hive **VIEW** `mnaas.ra_calldate_cust_monthly_rep` with columns: <br>`callmonth, tcl_secs_id, orgname, med_*_cdr, rej_*_cdr, gen_*_cdr, med_*_usage, rej_*_usage, gen_*_usage` (where `*` = data, sms, a2p, voi). |
| **Side‑effects** | DDL – creates/overwrites the view. No data mutation. |
| **Assumptions** | • All source tables exist and contain the columns referenced (`callmonth`, `tcl_secs_id`, `usage_type`, `cdr_count`, `bytes_sec_sms`, `cdr_usage`, `status`, `bill_month`). <br>• `usage_type` values are limited to the four listed. <br>• `month` variable is a valid `YYYYMM` string matching `month_reports.bill_month`. <br>• `org_details.orgno` matches `tcl_secs_id`. |

---

### 4. Integration Points  

| Connected component | Relationship |
|----------------------|--------------|
| **`mnaas_ra_calldate_traffic_table`** | Populated by upstream mediation jobs that parse raw CDRs. |
| **`mnaas_ra_calldate_rej_tbl`** | Filled by rejection‑handling jobs (e.g., validation, fraud). |
| **`mnaas_ra_calldate_gen_succ_rej`** | Produced by generation‑success pipelines (e.g., billing record creation). |
| **`mnaas.month_reports`** | Provides the current billing month; other scripts schedule monthly runs and set the `month` variable. |
| **`mnaas.org_details`** | Master data for organization names; maintained by a separate master‑data management script. |
| **Down‑stream consumers** | Monthly billing, revenue assurance, and reporting jobs that query `ra_calldate_cust_monthly_rep`. Typical consumers include: <br>• `mnaas_month_billing.hql` (billing aggregation). <br>• Dashboard/BI extracts. |
| **Orchestration layer** | Usually invoked from an Airflow/Control-M task that sets `hiveconf month=...` and runs this DDL before the reporting jobs. |

---

### 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Missing/incorrect `month` variable** – view compiles but returns empty or wrong data. | Validate presence of `${hiveconf:month}` before execution; fail fast if not set. |
| **Schema drift** – source tables add/rename columns, breaking the view. | Add a CI test that runs `DESCRIBE` on each source table and asserts required columns exist. |
| **Large data volume** – full‑month aggregation can be slow or OOM. | Ensure underlying tables are partitioned by `callmonth`; add `/*+ MAPJOIN */` hints if needed; monitor Hive execution logs. |
| **Division by zero** – usage conversion divides by constants only, but future changes could introduce zero denominators. | Keep conversion constants in a central config and guard against zero. |
| **Orphan organization IDs** – `org_details` may lack a matching row, resulting in NULL `orgname`. | Periodically reconcile `tcl_secs_id` values against `org_details`; optionally LEFT JOIN with a default placeholder. |

---

### 6. Running / Debugging the Script  

**Typical execution (Airflow/CLI)**  

```bash
# Set the month (e.g., March 2024)
export HIVE_MONTH=202403

# Run the DDL
hive -hiveconf month=${HIVE_MONTH} -f mediation-ddls/mnaas_ra_calldate_cust_monthly_rep.hql
```

**Debug steps**

1. **Validate variable** – `hive -e "set month;"` should show the value.  
2. **Explain plan** – after view creation, run:  
   ```sql
   EXPLAIN SELECT * FROM mnaas.ra_calldate_cust_monthly_rep LIMIT 10;
   ```  
   to verify join order and map‑reduce stages.  
3. **Sample data check** –  
   ```sql
   SELECT callmonth, tcl_secs_id, orgname, med_data_cdr, rej_data_cdr, gen_data_cdr
   FROM mnaas.ra_calldate_cust_monthly_rep
   WHERE callmonth = ${month}
   LIMIT 20;
   ```  
4. **Log inspection** – Hive logs (`/tmp/hive.log`) will contain any syntax or missing‑column errors.  

---

### 7. External Configuration / Environment Dependencies  

| Item | Usage |
|------|-------|
| **Hive variable `month`** | Filters `month_reports.bill_month` and the two generation/rejection tables. Must be supplied at runtime. |
| **Database `mnaas`** | All referenced tables and the view reside in this schema; the script assumes the default DB is set to `mnaas` or fully qualified names are used (as in the script). |
| **Hive execution engine** | The script relies on standard Hive SQL; no external UDFs are referenced. |

---

### 8. Suggested Improvements (TODO)

1. **Add explicit partition pruning** – If `ra_calldate_traffic_table`, `ra_calldate_rej_tbl`, and `ra_calldate_gen_succ_rej` are partitioned by `callmonth`, rewrite the CTEs to include `WHERE callmonth = ${month}` to limit scanned data.  
2. **Document required Hive variables** – Insert a comment block at the top of the file describing the `month` variable, expected format, and default handling (e.g., abort if unset).  

---