**File:** `mediation-ddls\mnaas_billing_preactive_with_grace_sims.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.billing_preactive_with_grace_sims`.  
The view lists pre‑active SIMs that are still within their grace period, joining the master pre‑active SIM list (`preactive_sim_list`) with the monthly billing calendar (`month_billing`) and the product‑split reference table (`gen_sim_product_split`). Rows are filtered out for a hard‑coded set of `tcl_secs_id` values, for empty commercial offers, and for SIMs already marked as activated.

The view is consumed downstream by billing‑pipeline jobs that need to know which pre‑active SIMs are still eligible for billing after the grace window.

---

## 2. Key Objects Defined in the File
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_preactive_with_grace_sims` | **VIEW** (`mnaas` schema) | Exposes a filtered, enriched list of pre‑active SIMs that are still within their grace period and not yet activated. |
| `createtab_stmt` | **SQL variable** (used by the build framework) | Holds the `CREATE VIEW` DDL; the framework substitutes it into the appropriate execution step. |

*No procedural code, classes, or functions are present – the file is a pure DDL definition.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input tables / sources** | `mnaas.preactive_sim_list` (alias `s`), `mnaas.month_billing`, `mnaas.gen_sim_product_split` (alias `p`). |
| **Columns referenced** | `s.tcl_secs_id`, `s.commercialoffer`, `s.sim`, `s.month`, `s.pre_act_date`, `s.grace_period`, `s.activated`; `month_billing.month`, `month_billing.end_with_time`; `p.secs_code`, `p.proposition`, `p.product_code`. |
| **Output** | Hive/Impala view `mnaas.billing_preactive_with_grace_sims`. |
| **Side‑effects** | DDL operation – creates or replaces the view in the `mnaas` database. No data mutation occurs. |
| **Assumptions** | • All referenced tables exist and are refreshed before this view is used.<br>• `product_code = 'SMC'` filter is valid for the current product catalog.<br>• `grace_period` may be `NULL`; `NVL(grace_period,0)` safely defaults to zero.<br>• The list `(41218, 41648, 37226)` represents known problematic `tcl_secs_id`s that must be excluded.<br>• `pre_act_date` is stored in a format castable to `TIMESTAMP`. |
| **External services** | Typically executed by a Hive/Impala client (e.g., `beeline`, `hive-cli`, or an Airflow/Hadoop job). No direct network calls, SFTP, or API interactions. |

---

## 4. Connections to Other Scripts & Components  

| Connected Component | Relationship |
|---------------------|--------------|
| `mnaas_billing_preactive_sims.hql` | Provides the base `preactive_sim_list` table that this view joins to. |
| `mnaas_billing_active_sims.hql` / `mnaas_billing_active_non_tolling_sims.hql` | Consume the view (or its underlying tables) to determine when a pre‑active SIM becomes active for billing. |
| Down‑stream ETL jobs (e.g., nightly billing aggregation) | Query `billing_preactive_with_grace_sims` to include eligible SIMs in revenue calculations. |
| `mnaas_billing_port_count.hql` | May reference the same `tcl_secs_id` set for port‑count metrics; any change in excluded IDs should be synchronized. |
| Build/Deployment framework (e.g., `run_ddl.sh` or Airflow DAG) | Wraps the `createtab_stmt` variable into a `hive -e "$createtab_stmt"` call. |

*Because the view is part of the “pre‑active” family, any schema change to `preactive_sim_list` or `gen_sim_product_split` will require coordinated updates across all related DDL scripts.*

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale underlying tables** – If `preactive_sim_list` or `month_billing` are not refreshed before the view is queried, results may be inaccurate. | Billing mis‑allocation, revenue leakage. | Enforce upstream refresh order in the DAG; add a health‑check that verifies `max(update_ts)` on source tables before downstream jobs run. |
| **Performance degradation** – The view performs a `LEFT OUTER JOIN` and a `NOT IN` filter on a potentially large table. | Long query runtimes, resource contention. | Verify that `tcl_secs_id`, `month`, and `secs_code` columns are bucketed/partitioned; consider materializing the view as a table with periodic refresh if query latency exceeds SLA. |
| **Hard‑coded exclusion list** – `(41218, 41648, 37226)` is embedded in the DDL. | Future changes to exclusion criteria require code change and redeployment. | Externalize the list to a configuration table (`mnaas.excluded_secs_ids`) and join instead of `NOT IN`. |
| **Null handling for `grace_period`** – If `grace_period` is unexpectedly non‑numeric, `NVL` may mask data issues. | Incorrect grace‑window calculation. | Add a validation step in the upstream load that enforces numeric type and logs anomalies. |
| **Schema drift** – Adding/removing columns in source tables without updating the view can cause query failures. | Job failures, downstream impact. | Include a schema‑validation test in CI that parses the view DDL and checks column existence. |

---

## 6. Example Execution & Debugging Workflow  

1. **Run the DDL** (typically part of a nightly deployment):  
   ```bash
   hive -e "$(cat mediation-ddls/mnaas_billing_preactive_with_grace_sims.hql | grep -v '^+')"   # or via Airflow task
   ```
   *The build framework extracts the `createtab_stmt` variable and executes it.*

2. **Validate creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.billing_preactive_with_grace_sims;
   ```

3. **Sample data check** (run as the operator):  
   ```sql
   SELECT COUNT(*) AS total_rows,
          SUM(CASE WHEN activated='N' THEN 1 ELSE 0 END) AS not_activated
   FROM mnaas.billing_preactive_with_grace_sims;
   ```

4. **Debugging a failure**:  
   - Check Hive/Impala logs for syntax errors.  
   - Verify that all referenced tables exist: `SHOW TABLES LIKE 'preactive_sim_list';` etc.  
   - Run the underlying SELECT manually (copy the SELECT clause) to isolate problematic rows.  
   - If the view returns zero rows unexpectedly, confirm that `pre_act_date` and `grace_period` values are populated correctly.

---

## 7. Configuration / Environment References  

| Reference | Usage |
|-----------|-------|
| `mnaas` database name | Hard‑coded in the DDL; may be overridden by a deployment‑time variable in the wrapper script. |
| Excluded `tcl_secs_id` list | Currently embedded; could be externalized to a config table or an environment variable (`EXCLUDED_SECS_IDS`). |
| Hive/Impala connection parameters (e.g., `hive.metastore.uris`) | Managed by the execution environment; not referenced directly in this file. |
| Logging / audit framework | Not part of the DDL; downstream jobs that query the view should emit audit records. |

---

## 8. Suggested TODO / Improvements  

1. **Externalize exclusion list** – Create a small reference table `mnaas.excluded_secs_ids` and replace the hard‑coded `NOT IN (…)` with a `LEFT ANTI JOIN`. This makes the exclusion criteria maintainable without code changes.  

2. **Add view comment & column documentation** – Include a `COMMENT` clause on the view and each selected column to aid downstream developers and data‑catalog tools (e.g., Hive Metastore, DataHub).  

---