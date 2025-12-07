**File:** `mediation-ddls\mnaas_billing_active_sims.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view `mnaas.billing_active_sims`. The view joins the `mnaas.active_sim_list` table with the `mnaas.month_billing` table on the `month` column, filters out three hard‑coded `tcl_secs_id` values, and excludes rows where `commercialoffer` is empty. The resulting view supplies downstream billing processes with a curated list of SIM identifiers (`tcl_secs_id`), their commercial offer, and the SIM number itself.

---

## 2. Core Artifact

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `createtab_stmt` | Hive DDL statement (view definition) | Defines the logical view `mnaas.billing_active_sims` that presents active SIMs ready for billing, applying business‑rule filters. |

*No procedural code, classes, or functions are present in this file; the entire file is a single DDL statement.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs (source tables)** | `mnaas.active_sim_list` (must contain columns `tcl_secs_id`, `commercialoffer`, `sim`, `month`) <br> `mnaas.month_billing` (must contain column `month`) |
| **Output** | Hive **view** `mnaas.billing_active_sims` (columns: `tcl_secs_id`, `commercialoffer`, `sim`) |
| **Side Effects** | - Registers/overwrites the view in the Hive metastore. <br> - No data is materialised; the view is evaluated at query time. |
| **Assumptions** | - Hive/Tez/MapReduce execution environment is available. <br> - The `mnaas` database exists and is accessible to the user running the script. <br> - Source tables are refreshed/maintained by upstream scripts (e.g., `mnaas_active_sim_list.hql`, `mnaas_month_billing.hql`). <br> - The three excluded `tcl_secs_id` values are static business exceptions. |
| **External Services** | Hive Metastore, Hadoop Distributed File System (HDFS) for underlying table data. No direct network calls, SFTP, or external APIs. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas.active_sim_list`** (created by `mnaas_active_sim_list.hql`) | Provides the base list of SIMs with their commercial offers. |
| **`mnaas.month_billing`** (likely created by a monthly‑billing DDL script) | Supplies the billing period (`month`) used to align SIM activity with the correct billing cycle. |
| **Downstream billing jobs** (e.g., nightly charge‑generation scripts) | Consume `mnaas.billing_active_sims` to calculate usage fees, generate invoices, or feed external billing platforms. |
| **Data quality / audit scripts** (e.g., `mnaas_billing_active_non_tolling_sims.hql`) | May join against this view to validate that only allowed SIMs are billed. |
| **Configuration / orchestration layer** (Airflow, Oozie, or custom scheduler) | Executes this HQL as part of a DAG that builds the “billing ready” data mart each month. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source‑table schema drift** (e.g., column rename or missing `commercialoffer`) | View creation fails; downstream billing breaks. | Add a pre‑execution validation step (`DESCRIBE` the source tables) and fail fast with a clear error message. |
| **Hard‑coded exclusion list becomes stale** | Incorrect SIMs may be billed or omitted. | Externalize the exclusion list to a configuration table or property file; reload dynamically. |
| **Large join causing performance degradation** (especially if `month_billing` is not partitioned) | Slow view resolution, possible OOM in Hive/Tez. | Ensure `month_billing` is small (one row per month) or partitioned; add `/*+ MAPJOIN(active_sim_list) */` hint if appropriate. |
| **View recreation overwrites existing view without versioning** | Unexpected changes for consumers that rely on a stable definition. | Use `CREATE OR REPLACE VIEW` only after a successful test run; optionally version the view name (`billing_active_sims_v20251204`). |
| **Missing database/permissions** | Script aborts with “database does not exist” or “access denied”. | Verify that the execution user has `CREATE`/`ALTER` rights on `mnaas`. Include a check at the start of the DAG. |

---

## 6. Execution & Debugging Guide

1. **Run the script** (typical in a scheduled job):  
   ```bash
   hive -f mediation-ddls/mnaas_billing_active_sims.hql
   ```
   *If using Beeline:*  
   ```bash
   beeline -u jdbc:hive2://<hive-host>:10000/mnaas -f mediation-ddls/mnaas_billing_active_sims.hql
   ```

2. **Validate creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.billing_active_sims;
   DESCRIBE FORMATTED mnaas.billing_active_sims;
   ```

3. **Test the view logic** (sample query):  
   ```sql
   SELECT COUNT(*) AS cnt, COUNT(DISTINCT tcl_secs_id) AS uniq_sims
   FROM mnaas.billing_active_sims;
   ```

4. **Debugging tips**  
   - If the view fails to create, run the inner SELECT alone to isolate errors:  
     ```sql
     SELECT tcl_secs_id, commercialoffer, sim
     FROM mnaas.active_sim_list a
     JOIN mnaas.month_billing b ON a.month = b.month
     WHERE tcl_secs_id NOT IN (41218,41648,37226)
       AND commercialoffer <> '';
     ```  
   - Check Hive logs (`/tmp/hive.log` or YARN application UI) for stack traces.  
   - Verify that the `month` column values match between the two tables (e.g., `SELECT DISTINCT month FROM ...`).  

---

## 7. Configuration & External Dependencies

| Item | Usage |
|------|-------|
| **Hive connection parameters** (`HIVE_HOST`, `HIVE_PORT`, `HIVE_USER`, `HIVE_PASSWORD`) | Required by the orchestration layer to launch the script. |
| **Database name** (`mnaas`) | Hard‑coded in the script; must exist. |
| **Excluded `tcl_secs_id` list** | Currently inlined (`(41218, 41648, 37226)`). Consider moving to a config table `mnaas.excluded_tcl_secs` or an environment variable (`EXCLUDED_TCL_SECS`). |
| **Logging / monitoring** | Not defined in the script; rely on Hive/Orchestration logs. |

---

## 8. Suggested Improvements (TODO)

1. **Externalize the exclusion list** – create a small reference table (`mnaas.excluded_tcl_secs`) and rewrite the filter as `tcl_secs_id NOT IN (SELECT tcl_secs_id FROM mnaas.excluded_tcl_secs)`. This makes the list maintainable without code changes.

2. **Add a defensive schema check** – prepend the script with a small validation block that aborts if required columns are missing, e.g.:  
   ```sql
   DESCRIBE mnaas.active_sim_list;
   DESCRIBE mnaas.month_billing;
   -- verify presence of tcl_secs_id, commercialoffer, sim, month
   ```

--- 

*End of documentation.*