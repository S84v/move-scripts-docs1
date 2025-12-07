**File:** `mediation-ddls\mnaas_billing_traffic_actusage.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.billing_traffic_actusage`. The view aggregates raw CDR records from `mnaas.traffic_details_billing` into a per‑month, per‑SIM, per‑traffic‑type summary that normalises usage units (KB, 100 KB, 30 sec blocks) and produces counts for inbound/outbound, domestic/international/roaming, and “MO/MT” directions. Down‑stream billing, reporting, and reconciliation jobs consume this view to calculate charges and generate invoices.

---

## 2. Important Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_traffic_actusage` | **View** | Provides a pre‑aggregated, month‑level snapshot of usage per SIM and related dimensions, ready for billing calculations. |
| `cdr` (CTE) | **Derived table** | Normalises raw usage values (`actualusage`) into two derived metrics: `actualusage_kb_sec_sms` and `actualusage_100kb_30sec_sms` based on `calltype`. |
| `traffic_details_billing` | **Source table** | Holds the raw CDRs (call date, SIM, usage, call type, traffic type, destination, etc.) that feed the view. |

*No procedural code, classes, or functions are defined in this file; the entire logic resides in the SQL view definition.*

---

## 3. Data Flow

| Aspect | Details |
|--------|---------|
| **Inputs** | `mnaas.traffic_details_billing` – must contain columns: `calldate`, `tcl_secs_id`, `usagefor`, `proposition_addon`, `sim`, `calltype`, `traffictype`, `country`, `zoneid`, `sponsor`, `apn`, `destinationtype`, `tadigcode`, `filename`, `actualusage`. |
| **Outputs** | Hive/Impala view `mnaas.billing_traffic_actusage`. The view is materialised on demand (no physical table created). |
| **Side‑effects** | DDL operation: creates or replaces the view. No data mutation. |
| **Assumptions** | • `traffic_details_billing` is refreshed before this view is queried. <br>• `calltype` values are limited to `'Data'`, `'Voice'`, or others that use the raw `actualusage`. <br>• `traffictype` values are `'MO'` (mobile originated) or `'MT'` (mobile terminated). <br>• Destination types are `'DOMESTIC'`, `'INTERNATIONAL'`, or `'ROAMING'`. <br>• The underlying Hive/Impala engine supports `WITH` CTEs, `ceil()`, and string functions used. |

---

## 4. Integration Points

| Connected Component | How it connects |
|---------------------|-----------------|
| **Down‑stream billing scripts** (e.g., `mnaas_billing_*` views) | Join on `sim`, `callmonth`, `tcl_secs_id` to fetch aggregated usage for charge calculation. |
| **ETL orchestration layer** (Airflow, Oozie, custom scheduler) | Executes this DDL as part of the *“billing‑ddl‑refresh”* DAG/job before any nightly billing aggregation runs. |
| **Reporting dashboards** (Superset, Tableau) | Query the view directly for usage‑by‑month reports. |
| **Data quality checks** (e.g., `mnaas_billing_traffic_actusage_qc.hql`) | May run row‑count or checksum jobs against the view to validate that the aggregation matches source CDR counts. |
| **Configuration** | Database connection details (host, port, auth) are supplied by the environment (e.g., `HIVE_JDBC_URL`, `IMPALA_HOST`). The script itself does not reference external config files, but the execution wrapper does. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Performance degradation** – the view aggregates millions of rows on each query. | Long query times, possible time‑outs in downstream jobs. | • Partition `traffic_details_billing` by `callmonth` (or `calldate`). <br>• Encourage downstream jobs to use `WHERE callmonth = 'YYYYMM'` filters. <br>• Collect table statistics after each load. |
| **Schema drift** – source table column changes break the view. | Job failures, missing data. | Add a schema‑validation step in the orchestration pipeline (e.g., `DESCRIBE` check). |
| **Incorrect unit conversion** – division by zero or unexpected `calltype` values. | Wrong usage totals, billing errors. | Guard against null/zero `actualusage` with `COALESCE(actualusage,0)`. Add a sanity‑check job that flags negative or extreme values. |
| **Stale view definition** – view not refreshed after source table alteration. | Inconsistent reports. | Use `CREATE OR REPLACE VIEW` (already done) and ensure the DDL job runs after every source table load. |
| **Security / data leakage** – view may expose raw usage to unauthorized users. | Compliance breach. | Apply Hive/Impala ACLs to restrict view access to billing service accounts only. |

---

## 6. Running / Debugging the Script

1. **Execution** (typically via a scheduler):  
   ```bash
   hive -f mediation-ddls/mnaas_billing_traffic_actusage.hql
   # or
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_billing_traffic_actusage.hql
   ```
2. **Verify creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.billing_traffic_actusage;
   ```
3. **Sample query** (quick sanity check):  
   ```sql
   SELECT callmonth, sim, SUM(actualusage_kb_sec_sms) AS total_kb
   FROM mnaas.billing_traffic_actusage
   WHERE callmonth = '202312'
   GROUP BY callmonth, sim
   LIMIT 10;
   ```
4. **Debug steps if the view fails to compile**:  
   - Run the inner CTE (`SELECT * FROM mnaas.traffic_details_billing LIMIT 5;`) to confirm column names/types.  
   - Check for nulls or unexpected `calltype` values that could cause `ceil()` errors.  
   - Look at Hive/Impala logs for syntax errors (often caused by missing backticks or mismatched parentheses).  

5. **Performance troubleshooting**:  
   - Run `EXPLAIN` on a representative query against the view.  
   - Verify that `traffic_details_billing` is partitioned and that statistics are up‑to‑date (`ANALYZE TABLE traffic_details_billing COMPUTE STATISTICS;`).  

---

## 7. External Configuration / Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_JDBC_URL` / `IMPALA_HOST` | Connection endpoint used by the orchestration wrapper that invokes this HQL file. |
| `HIVE_CONF_DIR` (or similar) | Provides Hive/Impala client configuration (Kerberos tickets, SSL, etc.). |
| No in‑script config references | The script relies entirely on the execution environment for DB connectivity and does not read external property files. |

---

## 8. Suggested Improvements (TODO)

1. **Add defensive handling for missing/NULL `actualusage`** – wrap the division expressions with `COALESCE(actualusage,0)` to avoid unexpected `NULL` results.  
2. **Document partitioning strategy** – include a comment block at the top of the file describing the expected partition columns of `traffic_details_billing` and recommended `SET hive.exec.dynamic.partition=true;` settings to ensure downstream queries are efficient.  

---