**File:** `mediation-ddls\mnaas_billing_forked_traffic_retusage.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view **`mnaas.billing_forked_traffic_retusage`**.  
The view aggregates raw CDR records from `mnaas.billing_traffic_forked_cdr` together with the month‑mapping table `mnaas.month_billing`. For each combination of call month, service element, usage type, SIM, call direction, traffic type, country, leg, call date, destination type, TADIG, and source filename it computes total usage, CDR counts, and a set of derived metrics (in/out usage, domestic vs. international/roaming usage, etc.). The view is used downstream by billing, reporting, and reconciliation jobs that need per‑month, per‑service usage summaries for “forked traffic” (i.e., traffic that has been split into separate CDR streams).

---

## 2. Key Objects Defined in the Script
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_forked_traffic_retusage` | **VIEW** (in schema `mnaas`) | Provides a pre‑aggregated, month‑level usage snapshot for forked traffic, exposing both raw totals and derived “in/out”, “domestic”, “international/roaming” metrics. |
| Columns (selected in the view) | – | `callmonth` (YYYY‑MM), `tcl_secs_id`, `usagefor`, `proposition_addon`, `sim`, `calltype`, `traffictype`, `country`, `leg`, `calldate`, `destinationtype`, `tadig`, `filename`, `usage`, `cdr_count`, `in_usage`, `in_cdr`, `out_usage`, `out_cdr`, `in_dom_usage`, `in_dom_cdr`, `in_int_rom_usage`, `in_int_rom_cdr`, `out_dom_usage`, `out_dom_cdr`, `out_int_rom_usage`, `out_int_rom_cdr`, `rom_count` – each representing a specific aggregation or count needed for downstream billing logic. |

*No procedural code, classes, or functions are defined – the file is pure DDL.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input tables** | `mnaas.billing_traffic_forked_cdr` (raw CDRs) <br> `mnaas.month_billing` (mapping of `bill_month` → `month` string) |
| **Output object** | Hive/Impala **VIEW** `mnaas.billing_forked_traffic_retusage` (persisted in the metastore) |
| **Side‑effects** | - Registers/overwrites the view definition in the metastore. <br> - No data is written or modified; only metadata changes. |
| **Assumptions** | - Both source tables exist, are refreshed before this script runs, and contain the columns referenced. <br> - `bill_month` in `billing_traffic_forked_cdr` matches the `month` column in `month_billing`. <br> - Data types allow the `SUM` and `COUNT` aggregations (numeric usage, integer CDR count). <br> - The Hive/Impala session has `CREATE VIEW` privileges on schema `mnaas`. |
| **External services** | None directly; the script is executed against the Hive/Impala cluster that hosts the `mnaas` database. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_billing_forked_traffic_actusage.hql`** (sibling script) | Likely creates a similar view for *actual* usage; downstream jobs may join the “retusage” (retained) view with the “actusage” view for reconciliation. |
| **ETL pipelines that populate `billing_traffic_forked_cdr`** | Must run **before** this view is (re)created, otherwise the view will reflect stale or empty data. |
| **Monthly billing jobs** (e.g., `mnaas_billing_active_sims.hql`, `mnaas_billing_api_transactions.hql`) | Consume this view to compute charges, generate invoices, or feed downstream reporting layers. |
| **Reporting / BI tools** (e.g., Tableau, PowerBI) | Query the view directly for dashboards on traffic volume, roaming usage, etc. |
| **Orchestration framework** (Airflow, Oozie, Control-M) | Executes this DDL as a step in the “billing‑aggregation” DAG/flow, typically after the raw CDR load step and before the charge‑calculation step. |
| **Metastore / Hive catalog** | The view definition is stored here; any schema change to source tables requires a view refresh (DROP/CREATE). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Performance degradation** – the view aggregates many columns and performs multiple conditional `SUM`s, which can be expensive on large CDR volumes. | Slow downstream queries, possible OOM errors. | • Ensure `billing_traffic_forked_cdr` is partitioned (e.g., by `bill_month` or `calldate`). <br>• Create materialized view or incremental summary tables if latency becomes unacceptable. |
| **Schema drift** – changes to source table column names/types break the view definition. | Job failures, missing data. | • Add a version‑controlled schema validation step before view creation. <br>• Keep the view definition under source control with explicit change‑log. |
| **Month‑mapping mismatch** – `bill_month` not present in `month_billing` leads to rows being dropped. | Incomplete usage data for a month. | • Validate that `SELECT COUNT(*) FROM month_billing WHERE month = <expected>` > 0 before view creation. |
| **Permission issues** – insufficient rights to CREATE VIEW. | Deployment halt. | • Include a pre‑flight check for `SHOW GRANT` on schema `mnaas`. |
| **Stale data** – view is not refreshed after a late‑arriving CDR batch. | Under‑billing or reporting errors. | • Schedule view recreation after the final CDR load window, or use a “refresh‑on‑demand” pattern. |

---

## 6. Execution & Debugging Guide  

1. **Typical run (via orchestration)**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_forked_traffic_retusage.hql
   # or
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_billing_forked_traffic_retusage.hql
   ```
   The orchestration tool should capture the exit code; a non‑zero code indicates a syntax or permission error.

2. **Manual verification**  
   ```sql
   -- Verify view exists
   SHOW CREATE VIEW mnaas.billing_forked_traffic_retusage;

   -- Sample query (limit 10)
   SELECT * FROM mnaas.billing_forked_traffic_retusage LIMIT 10;
   ```
   Check that `callmonth` values are in `YYYY‑MM` format and that aggregate columns are non‑null.

3. **Debugging steps if the view fails to create**  
   - Run the SELECT part alone (without `CREATE VIEW`) to see if any column reference errors appear.  
   - Verify that both source tables are accessible: `SELECT COUNT(*) FROM mnaas.billing_traffic_forked_cdr LIMIT 1;`  
   - Confirm that `month_billing` contains the expected mapping for the current billing month.  
   - Look at Hive/Impala logs for “SemanticException” or “AnalysisException” messages.

4. **Performance troubleshooting**  
   - Run `EXPLAIN` on the SELECT statement to see join and aggregation strategies.  
   - Check table statistics (`ANALYZE TABLE … COMPUTE STATISTICS`) for both source tables.  
   - Consider adding `PARTITION BY` on `callmonth` if the view is materialized later.

---

## 7. Configuration & External Dependencies  

| Item | Usage |
|------|-------|
| **Hive/Impala connection parameters** (e.g., `hive.metastore.uris`, `impala.host`) | Provided by the environment or orchestration config; not hard‑coded in the script. |
| **Schema name `mnaas`** | Assumed to be the target schema for all mediation objects. |
| **Environment variables** (if any) | None referenced directly in the script; however, the execution wrapper may inject variables for DB credentials or Kerberos tickets. |
| **External files** | None. The script is self‑contained DDL. |

---

## 8. Suggested Improvements (TODO)

1. **Add a comment block at the top of the file** describing purpose, owner, and version.  
2. **Convert the view to a materialized view or incremental summary table** (if supported) to improve query performance for downstream billing jobs, especially as CDR volume grows.  

---