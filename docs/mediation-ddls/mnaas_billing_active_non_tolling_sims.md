**File:** `mediation-ddls\mnaas_billing_active_non_tolling_sims.hql`

---

### 1. Summary
This Hive DDL script creates (or replaces) the view `mnaas.billing_active_non_tolling_sims`. The view joins the daily active‑SIM list with the month‑billing reference table, filters out any SIMs that appear in the tolling‑SIM list, excludes three hard‑coded `tcl_secs_id` values, and retains only rows where a commercial offer is present. The resulting view is used downstream for billing calculations that must ignore toll‑based SIMs and specific service‑center IDs.

---

### 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_active_non_tolling_sims` | Hive **VIEW** (in schema `mnaas`) | Provides a filtered set of active SIMs (with commercial offers) that are **not** part of the tolling SIM list and are not associated with excluded `tcl_secs_id`s. |
| `createtab_stmt` | Variable (used by the orchestration framework) | Holds the full `CREATE VIEW` statement; the framework extracts this variable to execute the DDL. |

*No procedural code, classes, or functions are present in this file.*

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs (source objects)** | - `mnaas.active_sim_list` (columns: `tcl_secs_id`, `commercialoffer`, `sim`, `month`)<br>- `mnaas.month_billing` (column: `month`)<br>- `mnaas.tolling_sim_list` (column: `sim`) |
| **Outputs** | - Hive view `mnaas.billing_active_non_tolling_sims` (columns: `tcl_secs_id`, `commercialoffer`, `sim`) |
| **Side Effects** | - Overwrites the view if it already exists.<br>- No data is written to tables; only metadata is altered. |
| **Assumptions** | - All referenced tables/views exist and are refreshed before this script runs.<br>- Column names and data types match those used in the SELECT clause.<br>- The three excluded `tcl_secs_id` values (41218, 41648, 37226) are static for the current release. |
| **External Services** | - Hive/Impala metastore (accessed via the orchestration engine).<br>- No direct SFTP, API, or queue interaction in this script. |

---

### 4. Integration Points

| Direction | Connected Component | How the Connection Is Made |
|-----------|---------------------|----------------------------|
| **Upstream** | `mnaas_active_sim_list.hql` | Populates `active_sim_list` used in the join. |
|  | `mnaas_month_billing.hql` (or similar) | Supplies `month_billing` reference data. |
|  | `mnaas_tolling_sim_list.hql` | Supplies the blacklist of SIMs to exclude. |
| **Downstream** | Billing aggregation scripts (e.g., `mnaas_billing_active_non_tolling_eids.hql`, `mnaas_billing_active_eids.hql`) | Consume the view to calculate non‑tolling billing metrics. |
|  | Reporting / KPI jobs | Query the view for dashboards or SLA checks. |
| **Orchestration** | Scheduler (Airflow, Oozie, or custom ETL runner) | Executes the script after the upstream tables are refreshed and before downstream billing jobs. The scheduler typically reads the `createtab_stmt` variable and runs it via Hive CLI / Beeline. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing source tables** | View creation fails, downstream jobs break. | Add a pre‑execution validation step that checks existence and schema of `active_sim_list`, `month_billing`, and `tolling_sim_list`. |
| **Schema drift** (e.g., column renamed) | Runtime SQL error, silent view loss. | Version‑control table schemas; include a schema‑validation test in CI pipeline. |
| **Hard‑coded exclusion list** becomes outdated | Incorrect SIMs may be billed or omitted. | Externalize the list to a configuration table (`mnaas.excluded_tcl_secs`) and join against it. |
| **Empty `commercialoffer` values** slipping through due to data quality issues | Billing may include non‑offer SIMs. | Add a data‑quality check that flags rows with `commercialoffer = ''` before view creation. |
| **View staleness** if upstream tables are refreshed after this script runs. | Downstream jobs use outdated data. | Ensure proper DAG ordering: this script must run **after** the latest refresh of its source tables. |

---

### 6. Execution / Debugging Guide

1. **Run the script** (typically via the orchestration engine):  
   ```bash
   hive -e "$(cat mediation-ddls/mnaas_billing_active_non_tolling_sims.hql | grep -v '^+' )"
   ```
   *The orchestration tool usually extracts the `createtab_stmt` variable and passes it to Hive/Beeline.*

2. **Verify creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.billing_active_non_tolling_sims;
   ```

3. **Quick sanity check** (row count & sample):  
   ```sql
   SELECT COUNT(*) FROM mnaas.billing_active_non_tolling_sims;
   SELECT * FROM mnaas.billing_active_non_tolling_sims LIMIT 10;
   ```

4. **Debugging failures**:  
   - Check Hive logs for “Table not found” or “Invalid column” errors.  
   - Run the SELECT part of the view manually to isolate which join/filter is failing.  
   - Verify that the three excluded `tcl_secs_id` values are still relevant.

5. **Rollback** (if needed):  
   - Drop the view: `DROP VIEW IF EXISTS mnaas.billing_active_non_tolling_sims;`  
   - Re‑run the script after fixing the root cause.

---

### 7. External Configuration / Environment Variables

| Config Item | Usage |
|-------------|-------|
| Hive/Impala connection parameters (`hive.metastore.uris`, `hive.server2.thrift.url`, etc.) | Provided by the orchestration environment; not referenced directly in the file. |
| Optional: `EXCLUDED_TCL_SECS` (if future improvement externalizes the list) | Would replace the hard‑coded `NOT IN (41218, 41648, 37226)` clause. |
| Logging / audit variables (e.g., `RUN_ID`) | May be injected by the scheduler for traceability, but not present in this script. |

---

### 8. Suggested Improvements (TODO)

1. **Externalize the exclusion list** – Create a small reference table (`mnaas.excluded_tcl_secs`) and replace the hard‑coded `NOT IN (…)` with a join/anti‑join. This makes the list maintainable without code changes.

2. **Add a header comment block** – Include script version, author, creation date, and a brief description of the view’s purpose. This aids future maintainers and supports automated documentation generation.