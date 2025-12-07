**File:** `mediation-ddls\mnaas_billing_preactive_sims.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view `mnaas.billing_preactive_sims`. The view joins the `preactive_sim_list` table with the `month_billing` table on the `month` column, filters out a hard‑coded list of `tcl_secs_id` values, excludes rows with an empty `commercialoffer`, and restricts the result set to records whose `pre_act_date` falls between the session variables `start_with_time` and `end_with_time`. The resulting view supplies a curated list of pre‑active SIM identifiers and their commercial offers for downstream billing and provisioning processes.

---

## 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `billing_preactive_sims` | Hive **VIEW** | Exposes a filtered, joined dataset of pre‑active SIMs (columns: `tcl_secs_id`, `commercialoffer`, `sim`). |
| `preactive_sim_list` | Hive **TABLE** | Source list of SIMs that are in a pre‑active state, includes `month`, `tcl_secs_id`, `commercialoffer`, `sim`, `pre_act_date`. |
| `month_billing` | Hive **TABLE** | Provides the billing calendar; used to align the `month` dimension with the SIM list. |
| `start_with_time` / `end_with_time` | Hive **SESSION VARIABLES** | Define the temporal window for `pre_act_date` filtering; set by the calling orchestration script. |

*No procedural code (functions, classes) is present; the file consists solely of a DDL statement.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Table `mnaas.preactive_sim_list` (must contain columns `month`, `tcl_secs_id`, `commercialoffer`, `sim`, `pre_act_date`).<br>- Table `mnaas.month_billing` (must contain column `month`).<br>- Hive session variables `start_with_time` and `end_with_time` (timestamp strings). |
| **Outputs** | - Hive view `mnaas.billing_preactive_sims` (columns `tcl_secs_id`, `commercialoffer`, `sim`). |
| **Side Effects** | - DDL operation: `CREATE VIEW`. If the view already exists, Hive will replace it (behavior depends on Hive version and `CREATE OR REPLACE VIEW` support). |
| **Assumptions** | - The database `mnaas` exists and is the active catalog.<br>- The excluded `tcl_secs_id` list (41218, 41648, 37226) is static for the current release.<br>- `commercialoffer` is a non‑null string when valid; empty string indicates an invalid record.<br>- `pre_act_date`, `start_with_time`, `end_with_time` are comparable (same datatype, e.g., `timestamp`). |

---

## 4. Integration Points  

| Consuming Component | How it Uses the View |
|---------------------|----------------------|
| **Billing aggregation jobs** (e.g., `mnaas_billing_*` scripts) | Join `billing_preactive_sims` to calculate pre‑active SIM revenue or to provision offers. |
| **Provisioning orchestration** (custom Python/Java ETL) | Query the view to trigger activation workflows for SIMs whose pre‑act date is now within the processing window. |
| **Reporting dashboards** (e.g., Tableau/PowerBI via Hive connector) | Visualize the count of pre‑active SIMs per commercial offer. |
| **Data quality checks** (e.g., `*_reject.hql` scripts) | May reference the same source tables to validate that excluded IDs are correctly filtered. |

*The view is a downstream data product; any script that needs a filtered list of pre‑active SIMs should reference `mnaas.billing_preactive_sims` rather than re‑implementing the join/filter logic.*

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or changed source tables/columns** | View creation fails; downstream jobs break. | Include a pre‑run schema validation step (e.g., `DESCRIBE` checks) in the orchestration. |
| **Hard‑coded exclusion list becomes stale** | Unwanted SIMs may be processed, causing billing errors. | Externalize the exclusion list to a configuration table (`mnaas.excluded_tcl_secs`) and join instead of `NOT IN`. |
| **Session variables not set or malformed** | `pre_act_date` filter may return all rows or none. | Add a guard clause: `WHERE ${hiveconf:start_with_time} IS NOT NULL AND ${hiveconf:end_with_time} IS NOT NULL`. |
| **View replacement without version control** | Unexpected schema changes affect downstream jobs. | Use `CREATE OR REPLACE VIEW` only after a successful `SELECT` validation; keep DDL under Git with version tags. |
| **Performance degradation on large joins** | Long view creation time, possible OOM. | Ensure `month_billing.month` is partitioned or indexed; consider materializing as a table with incremental refresh. |

---

## 6. Execution & Debugging Guide  

1. **Set required variables** (usually done by the orchestration script):  
   ```bash
   hive -hiveconf start_with_time='2025-01-01 00:00:00' \
        -hiveconf end_with_time='2025-01-31 23:59:59' \
        -f mediation-ddls/mnaas_billing_preactive_sims.hql
   ```

2. **Run the script** – the Hive CLI or Beeline will execute the `CREATE VIEW` statement.  
   - On success, Hive prints `OK` and the view appears in `SHOW VIEWS;`.

3. **Validate the view**:  
   ```sql
   SELECT COUNT(*) AS total, 
          COUNT(DISTINCT tcl_secs_id) AS uniq_sims
   FROM mnaas.billing_preactive_sims;
   ```
   - Compare counts with expectations from source tables.

4. **Debug failures**:  
   - **Syntax / variable errors** – Hive will report “Invalid column reference” or “Undefined variable”. Verify variable names and quoting.  
   - **Missing tables/columns** – Run `DESCRIBE mnaas.preactive_sim_list;` and `DESCRIBE mnaas.month_billing;` to confirm schema.  
   - **Empty result set** – Check that `start_with_time`/`end_with_time` correctly bound the `pre_act_date` range.

5. **Log collection** – Capture Hive logs (`hive.log` or Beeline stdout) for audit; include the executed DDL and variable values.

---

## 7. External Configuration / Environment Dependencies  

| Item | Source | Usage |
|------|--------|-------|
| `start_with_time` / `end_with_time` | Set by the calling batch/orchestration (e.g., Airflow, Oozie, custom shell script). | Controls the temporal filter on `pre_act_date`. |
| Excluded `tcl_secs_id` list | Hard‑coded in the DDL (`NOT IN (41218, 41648, 37226)`). | May be moved to a config table or external properties file for flexibility. |
| Hive metastore connection | Configured in `hive-site.xml` or environment variables (`HIVE_CONF_DIR`). | Required for any DDL execution. |

If the orchestration framework uses a properties file (e.g., `billing.properties`), ensure the two time variables are defined there and passed via `-hiveconf`.

---

## 8. Suggested Improvements (TODO)

1. **Externalize the exclusion list** – replace the static `NOT IN (…)` clause with a join to a configuration table (`mnaas.excluded_tcl_secs`) so that additions/removals do not require code changes.  
2. **Add defensive variable checks** – prepend the script with a small HiveQL block that aborts if `start_with_time` or `end_with_time` are missing or not parsable, e.g.:  
   ```sql
   SET hivevar:missing_vars = '';
   IF (${hiveconf:start_with_time} IS NULL) SET hivevar:missing_vars = concat(${hivevar:missing_vars}, ' start_with_time');
   IF (${hiveconf:end_with_time} IS NULL)   SET hivevar:missing_vars = concat(${hivevar:missing_vars}, ' end_with_time');
   IF (${hivevar:missing_vars} != '') THEN
       SELECT concat('Missing required variables:', ${hivevar:missing_vars}) AS error_msg;
       EXIT;
   END IF;
   ```

Implementing these changes will reduce operational friction and improve maintainability.