**File:** `mediation-ddls\mnaas_billing_suspended_with_grace_sims.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.billing_suspended_with_grace_sims`. The view consolidates suspended SIM records with month‑billing information and, when available, product‑split data for the “SSC” product. It filters out a hard‑coded list of TCL SEC IDs, applies the grace‑period logic (`suspended_date + grace_period ≤ end_with_time`), excludes empty commercial offers, and keeps only SIMs that have not yet been activated.

---

## 2. Core Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `billing_suspended_with_grace_sims` | **View** | Provides a ready‑to‑query dataset of suspended SIMs that are still within their grace period, enriched with month‑billing and product‑split attributes. |
| `mnaas.suspended_sim_list` | Table | Source of suspended SIM records (`tcl_secs_id`, `commercialoffer`, `sim`, `month`, `suspended_date`, `grace_period`, `activated`). |
| `mnaas.month_billing` | Table | Supplies the billing month reference (`month`, `end_with_time`). |
| `mnaas.gen_sim_product_split` | Table | Optional join to retrieve the product code (`product_code`) for a given SIM (`secs_code`, `proposition`). Only rows where `product_code = 'SSC'` are considered. |

*No procedural code, classes, or functions are defined in this file; the only “executable” element is the `CREATE VIEW` DDL statement.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.suspended_sim_list` (must contain columns used in the SELECT).<br>- `mnaas.month_billing` (must contain `month` and `end_with_time`).<br>- `mnaas.gen_sim_product_split` (optional; must contain `secs_code`, `proposition`, `product_code`). |
| **Outputs** | - Hive/Impala view `mnaas.billing_suspended_with_grace_sims` (columns: `tcl_secs_id`, `commercialoffer`, `sim`). |
| **Side Effects** | - Overwrites the view if it already exists (standard Hive `CREATE VIEW` semantics). No data mutation occurs on source tables. |
| **Assumptions** | - All referenced tables exist and are up‑to‑date for the billing cycle being processed.<br>- `suspended_date` is a timestamp‑compatible column.<br>- `grace_period` is numeric (days) and may be NULL; `NVL(grace_period,0)` handles that.<br>- The list `(41218, 41648, 37226)` represents TCL SEC IDs that must be excluded for business reasons.<br>- The environment runs Hive/Impala with support for `ADD_MONTHS` and `NVL` functions. |

---

## 4. Integration with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| `mnaas_billing_suspended_sims.hql` | Likely creates the base `suspended_sim_list` table used here. |
| `mnaas_billing_preactive_sims.hql` / `mnaas_billing_preactive_with_grace_sims.hql` | Parallel DDLs that generate views for pre‑active SIMs; they share similar join patterns and may be consumed together in downstream reporting jobs. |
| Billing ETL pipelines (e.g., nightly “billing‑aggregation” job) | The view is read by downstream aggregation scripts that compute revenue, usage, or provisioning metrics. |
| Monitoring / Alerting scripts | May query the view to detect SIMs that are about to exit the grace period without activation. |
| Data‑warehouse UI (e.g., Hue, Superset) | End‑users run ad‑hoc queries against the view for operational dashboards. |

*Because the view depends on `month_billing`, it must be refreshed (or the underlying tables refreshed) before the start of each billing month.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded exclusion list** (`tcl_secs_id NOT IN (…)`) becomes stale. | Incorrect SIMs may be included/excluded, leading to billing errors. | Externalize the list to a reference table (e.g., `mnaas.excluded_secs_ids`) and join instead of hard‑coding. |
| **Missing columns or schema drift** in any source table. | View creation fails; downstream jobs break. | Add a pre‑deployment validation step that checks column existence and data types. |
| **Performance degradation** due to full table scans on large `suspended_sim_list`. | Slow downstream queries, possible timeouts. | Ensure `suspended_sim_list` is partitioned by `month` and/or `activated`. Add appropriate statistics (`ANALYZE TABLE`). |
| **Grace‑period calculation mismatch** (different time‑zone handling). | SIMs may be incorrectly classified as still in grace period. | Confirm that `suspended_date` and `end_with_time` are stored in the same timezone; document the expected timezone. |
| **View becomes stale** if underlying tables are refreshed after view creation (e.g., incremental loads). | Queries return outdated data. | Use `CREATE OR REPLACE VIEW` each run, or schedule a view refresh after any load that modifies source tables. |

---

## 6. Running / Debugging the Script

1. **Execution** (typical CI/CD or manual run)  
   ```bash
   # Using Hive CLI
   hive -f mediation-ddls/mnaas_billing_suspended_with_grace_sims.hql

   # Or using Beeline (recommended)
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaas_billing_suspended_with_grace_sims.hql
   ```

2. **Verification**  
   ```sql
   SELECT COUNT(*) AS total_rows
   FROM   mnaas.billing_suspended_with_grace_sims;
   ```

3. **Debugging Tips**  
   - Run the SELECT part of the view directly with a `LIMIT` to inspect sample rows.  
   - If the view fails to create, check the Hive metastore logs for “Table not found” or “Invalid column” errors.  
   - Use `DESCRIBE FORMATTED mnaas.billing_suspended_with_grace_sims;` to confirm column types and view definition.  
   - Compare row counts against the source tables to ensure the exclusion list and grace‑period filter behave as expected.

---

## 7. External Configuration / Environment Variables

| Config / Env | Usage |
|--------------|-------|
| Hive/Impala connection parameters (`HIVE_HOST`, `HIVE_PORT`, `HIVE_USER`, `HIVE_PASSWORD`) | Required by the execution wrapper (CLI/Beeline) but not referenced inside the HQL file. |
| `HIVE_CONF_DIR` (optional) | May point to custom Hive configuration (e.g., metastore URIs). |
| No in‑file placeholders or `${}` variables are present; the script is self‑contained. |

If the organization uses a templating layer (e.g., Jinja, Airflow `HiveOperator`), those variables would be injected before the file is sent to Hive.

---

## 8. Suggested Improvements (TODO)

1. **Externalize the exclusion list** – replace the hard‑coded `NOT IN (41218, 41648, 37226)` with a join to a reference table (`mnaas.excluded_secs_ids`). This makes the list maintainable without code changes.

2. **Add a comment block** at the top of the file documenting:  
   - Purpose of the view.  
   - Expected input table schemas.  
   - Refresh cadence (e.g., “run nightly after month_billing load”).  
   - Owner / contact information.  

These changes improve maintainability and reduce the risk of silent data‑quality issues.