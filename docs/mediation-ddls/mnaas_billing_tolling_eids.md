**File:** `mediation-ddls\mnaas_billing_tolling_eids.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.billing_tolling_eids`. The view consolidates **current** and **late** toll‑charging identifiers (EIDs) for SIMs that appear in “HOL”‑type traffic files. It extracts distinct `tcl_secs_id`, commercial offer description, EID, and month of the call, and flags each record as billable (`'Yes'`/`'No'`). The view is later consumed by downstream billing, reconciliation, and reporting jobs that need a definitive list of toll‑eligible SIMs per month.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `curr_toll_eids` | CTE (sub‑query) | Pulls distinct EIDs from the **raw daily traffic** table for the current billing period, limited to files containing “HOL”. Marks every row as billable (`'Yes'`). |
| `late_toll_eids` | CTE (sub‑query) | Handles **late‑arriving CDRs** by joining `tolling_eid_list` (master list of known billable EIDs) with `traffic_details_billing_late_cdr`. Determines billability: if the EID exists in the master list → `'Yes'`, otherwise `'No'`. |
| `all_toll_sims` | CTE (union) | Merges the current and late sets, removing duplicates. |
| `billing_tolling_eids` | Hive **VIEW** | Exposes the final distinct set of `(callmonth, billable, tcl_secs_id, commercialoffer, eid)` for downstream consumption. |

*No procedural code, classes, or functions are defined – the file is pure DDL.*

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Input tables** | `mnaas.traffic_details_raw_daily` (raw CDRs), `mnaas.month_billing` (billing period metadata), `mnaas.tolling_eid_list` (master EID list), `mnaas.traffic_details_billing_late_cdr` (late CDRs). |
| **Key columns used** | `partition_date`, `start_date`, `end_date`, `filename`, `tcl_secs_id`, `commercialofferdescription`, `euiccid`, `calldate`, `bill_month`, `month`. |
| **External parameters** | Hive variables `start_date` and `end_date` are expected to be set in the session (usually by a preceding orchestration step that determines the billing window). |
| **Output** | Hive view `mnaas.billing_tolling_eids`. No physical tables are created or modified. |
| **Side effects** | Overwrites the view if it already exists (implicit `CREATE OR REPLACE VIEW` semantics in most Hive versions). |
| **Assumptions** | • All referenced tables exist and are refreshed before this script runs.<br>• `filename` contains the substring “HOL” for holiday‑related traffic only.<br>• `euiccid` may be NULL; the script treats NULL and empty string as “not present”.<br>• `month_billing` provides a single row per billing month with `start_date`/`end_date` covering the desired partition range. |

---

## 4. Integration Points  

| Connected component | How the view is used |
|---------------------|----------------------|
| **Downstream billing jobs** (e.g., `mnaas_billing_*` scripts) | Join `billing_tolling_eids` with usage, activation, or suspension tables to compute chargeable events per SIM. |
| **Reconciliation / audit scripts** | Validate that every “HOL” CDR has a corresponding entry in the view and that billable flags match expectations. |
| **Reporting dashboards** | Pull `callmonth`, `billable`, `commercialoffer`, `eid` for KPI calculations (e.g., holiday‑traffic revenue). |
| **Orchestration layer** (Airflow/Oozie) | Typically preceded by a task that sets Hive variables `start_date`/`end_date` based on the current month‑billing record. Followed by tasks that materialize aggregates from the view. |

*Because the view is schema‑only, any change to underlying tables (column rename, type change) will break downstream scripts that reference it.*

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale or missing billing window variables** (`start_date`/`end_date`) | View may return zero rows or wrong month range. | Validate that the variables are set (e.g., `SET hivevar:start_date;`) before execution; fail fast if missing. |
| **Full outer join performance** on `late_toll_eids` | Large late‑CDR volumes can cause long runtimes or OOM errors. | Ensure both sides are bucketed/partitioned on `eid`/`euiccid`; consider rewriting as left join if master list is exhaustive. |
| **Duplicate rows** after UNION | Downstream aggregates may double‑count. | Use `SELECT DISTINCT` (already present) but also verify that source tables have proper primary keys or deduplication logic. |
| **Schema drift** (e.g., `commercialofferdescription` renamed) | View creation fails, breaking the pipeline. | Add a schema‑validation step before view creation; keep DDL under version control with automated tests. |
| **Incorrect “HOL” filter** (filename pattern change) | Missing or extra records. | Centralise filename pattern in a config file or Hive variable; document the expected naming convention. |

---

## 6. Running / Debugging the Script  

1. **Preparation (usually done by the orchestrator)**  
   ```bash
   hive -e "SET start_date='2024-09-01'; SET end_date='2024-09-30';"
   ```
2. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_tolling_eids.hql
   ```
   - On Hive 2.x+ the command will automatically replace the view if it exists.  
   - Capture the exit code; non‑zero indicates a syntax or table‑resolution error.

3. **Validate the view**  
   ```sql
   SELECT COUNT(*) AS rows, 
          SUM(CASE WHEN billable='Yes' THEN 1 ELSE 0 END) AS billable_cnt
   FROM mnaas.billing_tolling_eids;
   ```
   - Compare counts with expectations from the source CDR tables.

4. **Debugging tips**  
   - Run each CTE individually (copy‑paste into a Hive session) to inspect intermediate results.  
   - Use `EXPLAIN` on the final SELECT to verify join order and partition pruning.  
   - Check Hive logs (`/tmp/hive.log` or YARN UI) for “MapReduce” task failures.

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| Hive variables `start_date`, `end_date` | Define the billing window for the `curr_toll_eids` CTE. |
| Table/partition locations (e.g., HDFS paths) | Implicitly required; the script assumes the tables are already created and partitioned by `partition_date` or `bill_month`. |
| Naming convention for “HOL” files | Hard‑coded `filename LIKE '%HOL%'`; any change must be reflected in the script or external config. |

---

## 8. Suggested Improvements (TODO)

1. **Parameterise the “HOL” pattern** – move `'%HOL%'` into a Hive variable (e.g., `hol_pattern`) so that a change in file‑naming does not require code modification.  
2. **Add explicit view replacement** – use `CREATE OR REPLACE VIEW` (if supported) or drop‑and‑create logic with a safety check to avoid accidental view loss during concurrent runs.  

---