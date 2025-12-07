**File:** `mediation-ddls\mnaas_v_traffic_details_billing_late_cdr.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.v_traffic_details_billing_late_cdr`.  
The view consolidates “late” CDR records from the raw billing table `traffic_details_billing_late_cdr` for a given billing month, applying a series of quality filters, de‑duplication (via `row_number()`), and business‑rule transformations. It produces a unified, ready‑to‑consume dataset that distinguishes Data/Voice usage, SMS‑MO/MT, and A2P traffic, while handling sponsor extraction, proposition add‑on resolution, and a “telesur reject” flag. Down‑stream aggregation, billing, and reporting jobs reference this view.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| **`v_traffic_details_billing_late_cdr`** | View | Exposes a cleaned, de‑duplicated set of late‑CDR rows for the current billing month, ready for downstream billing calculations. |
| **`filtered_cdr`** | CTE (Common Table Expression) | Pulls raw rows from `traffic_details_billing_late_cdr` (joined to `month_billing` for month filter) and applies all source‑level filters, column derivations, and the `row_number()` window to identify the latest version per `(cdrid, chargenumber)`. |
| **`traffic_details_billing_late_cdr`** | Source table | Holds raw late‑CDR records (post‑mediation) that have not yet been processed into the final billing view. |
| **`month_billing`** | Reference table | Provides the `bill_month` value used to restrict the view to a single month (`WHERE bill_month = \`month\``). |
| **`billing_a2p_calling_party`** | Reference table | Supplies the list of MSISDNs that are classified as A2P (Application‑to‑Person) callers; used to split SMS‑MO traffic into regular vs. A2P. |
| **Derived columns** | – | `sponsor` (first 5 digits of IMSI), `usage`/`actualusage` (decoded per calltype), `proposition_addon`, `usagefor`, `gen_filename`, `telesur_reject`, `partition_date`. |
| **UNION blocks** | – | Four logical blocks that emit rows for: (1) Data/Voice, (2) SMS‑MT, (3) SMS‑MO (non‑A2P), (4) SMS‑MO (A2P). Each block filters on `row_num = 1` and `telesur_reject = 'N'`. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • `mnaas.traffic_details_billing_late_cdr` (raw late CDRs) <br>• `mnaas.month_billing` (contains `bill_month` column) <br>• `mnaas.billing_a2p_calling_party` (list of excluded MSISDNs) |
| **External parameters** | Hive/Impala variable **`month`** (e.g., `-hiveconf month=202312`). The script expects this variable to be set at execution time. |
| **Outputs** | View `mnaas.v_traffic_details_billing_late_cdr`. No physical tables are created/modified; the view definition is stored in the metastore. |
| **Side Effects** | DDL operation – replaces the view definition. If the view already exists, it is overwritten. No data movement occurs. |
| **Assumptions** | • Source tables exist and have the columns referenced in the SELECT list. <br>• `month` variable is supplied and matches a value in `month_billing`. <br>• `filename` pattern contains “HOL” and not “NonMOVE”. <br>• `tcl_secs_id`, `usedtype`, `balancetypeid` values follow the documented business rules. |

---

## 4. Integration Points  

| Direction | Component | How it connects |
|-----------|-----------|-----------------|
| **Upstream** | Scripts that populate `traffic_details_billing_late_cdr` (e.g., `mnaas_traffic_details_billing_late_cdr.hql`). | Provide raw late‑CDR rows that this view consumes. |
| **Upstream** | `month_billing` loader (likely a daily/monthly dimension table). | Supplies the `bill_month` filter. |
| **Upstream** | `billing_a2p_calling_party` maintenance job. | Determines which SMS‑MO rows become A2P. |
| **Downstream** | Aggregation scripts such as `mnaas_traffic_details_billing_aggr_daily.hql` or billing charge‑generation jobs. | Query this view to calculate usage, revenue, and generate invoices. |
| **Downstream** | Reporting dashboards / BI tools that reference `v_traffic_details_billing_late_cdr`. | Provide operational visibility into late‑CDR processing. |
| **Scheduler** | Typically invoked by an orchestrator (Airflow, Oozie, Control-M) as part of the nightly “billing‑late‑CDR” pipeline. | Runs after the raw late‑CDR load and before the billing aggregation step. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source tables change (column added/removed, type change). | View creation fails or returns incorrect data. | Add automated schema‑validation step before view recreation; version‑control view definition alongside source schema. |
| **Missing/incorrect `month` variable** – view filters on a non‑existent month value. | View ends up empty or contains data from wrong month. | Enforce parameter presence in the orchestration layer; add a pre‑run check that `SELECT COUNT(*) FROM month_billing WHERE bill_month = '${month}'` > 0. |
| **Performance degradation** – large UNION + window function on massive CDR volume. | Long view creation time; downstream jobs may miss SLA. | Ensure `traffic_details_billing_late_cdr` is partitioned on `calldate`/`partition_date`; add appropriate statistics; consider materializing as a table with incremental loads if latency becomes critical. |
| **Incorrect A2P classification** – stale `billing_a2p_calling_party` data. | Mis‑billing of SMS traffic. | Refresh the A2P reference table daily; add a data‑quality alert if the row count changes unexpectedly. |
| **`telesur_reject` logic** – hard‑coded rule (`usedtype='2' AND tcl_secs_id=25050 AND balancetypeid != 221`). | Unexpected inclusion/exclusion of rows. | Document the rule; add unit‑test queries that verify a sample of known reject cases are filtered out. |

---

## 6. Running / Debugging the Script  

1. **Typical execution (via Hive/Impala CLI):**  
   ```bash
   hive -hiveconf month=202312 -f mediation-ddls/mnaas_v_traffic_details_billing_late_cdr.hql
   ```
   Or via Impala:
   ```bash
   impala-shell -i <impala-host> -q "SET month=202312; SOURCE mediation-ddls/mnaas_v_traffic_details_billing_late_cdr.hql;"
   ```

2. **Orchestrator example (Airflow DAG snippet):**  
   ```python
   HiveOperator(
       task_id='create_late_cdr_view',
       hql='mediation-ddls/mnaas_v_traffic_details_billing_late_cdr.hql',
       hive_cli_conn_id='hive_default',
       parameters={'month': '{{ ds_nodash[:6] }}'}   # e.g., 202312
   )
   ```

3. **Debug steps:**  
   - Verify the `month` variable is set: `SET month;`  
   - Run a preview query:  
     ```sql
     SELECT * FROM mnaas.v_traffic_details_billing_late_cdr LIMIT 100;
     ```  
   - Check row counts vs. source:  
     ```sql
     SELECT COUNT(*) FROM mnaas.traffic_details_billing_late_cdr WHERE bill_month = '${month}';
     SELECT COUNT(*) FROM mnaas.v_traffic_details_billing_late_cdr;
     ```  
   - If view creation fails, inspect the Hive/Impala log for syntax errors or missing columns.

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| **Hive/Impala variable `month`** | Filters the view to a single billing month (`WHERE bill_month = \`month\``). Must be supplied at runtime. |
| **Metastore connection** | The view definition is stored in the Hive/Impala metastore; proper permissions are required to CREATE/REPLACE VIEW. |
| **Database/user privileges** | The executing user must have `CREATE VIEW` rights on schema `mnaas` and SELECT rights on the three source tables. |
| **File naming conventions** | The script expects `filename` values containing “HOL” and not “NonMOVE”; this is part of the upstream file‑naming policy. |

---

## 8. Suggested Improvements (TODO)

1. **Materialize the view as an incremental table** – If downstream jobs experience latency, replace the view with a partitioned table that is refreshed nightly (INSERT OVERWRITE) to avoid re‑evaluating the heavy UNION/ROW_NUMBER logic on every query.  
2. **Externalize the hard‑coded filter constants** (e.g., `tcl_secs_id = 25050`, `balancetypeid != 221`, excluded MSISDN list) into a configuration table or property file, allowing business rule changes without code edits.

---