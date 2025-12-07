**File:** `mediation-ddls\mnaas_ra_calldate_geneva_reject.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.ra_calldate_geneva_reject`.  
The view surfaces rejected CDR (Call Detail Record) entries whose *proposition* does not exist in the Geneva product catalogue. It joins the reject‑record table (`ra_calldate_gen_succ_rej`) with the traffic table (`ra_calldate_traffic_table`) and enriches the result with organization details (`org_details`). For each distinct combination of file, processing date, month, SIM, organization, proposition, country and usage type it aggregates the number of CDRs and the total usage (converted to MB for data, minutes for voice, or raw units for other types) and tags the row with a fixed reason string.

---

## 2. Key Objects Defined / Used  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.ra_calldate_geneva_reject` | **VIEW** | Exposes aggregated reject‑record metrics for downstream reporting or further ETL steps. |
| `mnaas.ra_calldate_gen_succ_rej` | **TABLE** | Source of rejected CDRs (status = 'REJECTED') with columns: `filename`, `processed_date`, `secs_id`, `proposition`, `usage_type`, `callmonth`, `bill_month`, `status`, `row_num`. |
| `mnaas.ra_calldate_traffic_table` | **TABLE** | Holds raw traffic details (CDR count, bytes/sec/SMS, calltype, etc.) used to compute usage amounts. |
| `mnaas.org_details` | **TABLE** | Optional enrichment table mapping `tcl_secs_id` → `orgname` (joined via `orgno`). |
| `createtab_stmt` | **VARIABLE** (script placeholder) | Holds the DDL statement; the script prints it for downstream execution by the orchestration engine. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - `ra_calldate_gen_succ_rej` (filtered on `status='REJECTED'` and `row_num = 1`).<br>- `ra_calldate_traffic_table` (joined on `secs_id`, `proposition_addon`, `usage_type`, `callmonth`, `bill_month`).<br>- `org_details` (LEFT OUTER join on `tcl_secs_id = orgno`). |
| **Outputs** | - Persistent view `mnaas.ra_calldate_geneva_reject` (read‑only logical table). |
| **Side‑Effects** | - DDL execution that creates or replaces the view in the `mnaas` schema. No data mutation occurs. |
| **Assumptions** | - All referenced tables exist and have the columns used in the query.<br>- `tcl_secs_id` can be safely cast to STRING for the join.<br>- `row_num` is generated elsewhere (likely a window function) guaranteeing a single row per reject key.<br>- The usage types are limited to `'DAT','SMS','VOI','A2P'`.<br>- The Hive/Impala engine supports the `CASE` expression and the `sum()` aggregation as written. |

---

## 4. Integration Points (How This View Connects to the Rest of the System)

| Down‑stream Component | Expected Interaction |
|-----------------------|----------------------|
| **Aggregation Views** (e.g., `mnaas_ra_calldate_geneva_reject_aggr` or similar) | Consume this view to produce daily/monthly summary tables for reporting dashboards. |
| **ETL Jobs** (batch scripts that materialize fact tables) | Query the view to load reject‑record metrics into a data warehouse (e.g., `fact_ra_rejects`). |
| **Alerting / Monitoring** | A scheduled job may query the view for spikes in `cdr_count` or `cdr_usage` and raise alerts. |
| **Data Quality Checks** | Validation scripts compare counts in this view against source tables to ensure no lost rejects. |
| **External APIs** (e.g., a REST service exposing reject statistics) | The API layer reads from the view (or a materialized table derived from it) to serve UI dashboards. |

*Note:* The view name follows the same naming convention as other `ra_calldate_*_reject` views (e.g., `ra_calldate_country_reject`, `ra_calldate_duplicate_reject`). Those sibling views are likely used together in a union or in a master reject‑report.

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source tables change (column rename, type change) | View creation fails or returns wrong results | Add a pre‑deployment schema validation step (e.g., `DESCRIBE` checks) and version‑lock the view definition. |
| **Join cardinality explosion** – mismatched `secs_id` casting leads to many-to‑many joins | Performance degradation, inflated aggregates | Verify that `CAST(t.tcl_secs_id AS STRING)` matches the type of `r.secs_id`. Consider using explicit hash joins or adding indexes/partitions on join keys. |
| **Stale data** – view reflects only the latest batch if underlying tables are partitioned by date and not refreshed | Incomplete reporting | Ensure downstream jobs run after the source tables are fully loaded for the processing window. |
| **Incorrect usage conversion** – `bytes_sec_sms` division logic may be wrong for new call types | Mis‑reported usage metrics | Add unit tests for the `CASE` expression covering all supported `calltype` values. |
| **Missing org details** – LEFT OUTER join may return NULL orgname, causing downstream null‑handling issues | Data quality problems | Document that `orgname` can be NULL and enforce default handling in downstream consumers. |

---

## 6. Example Execution & Debugging Workflow  

1. **Run the script** (typically via the orchestration framework, e.g., Airflow, Oozie, or a custom scheduler):  
   ```bash
   hive -f mediation-ddls/mnaas_ra_calldate_geneva_reject.hql
   ```
   *or* the script may be embedded in a larger batch file that extracts the `createtab_stmt` variable and executes it.

2. **Verify view creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.ra_calldate_geneva_reject;
   SELECT COUNT(*) FROM mnaas.ra_calldate_geneva_reject LIMIT 10;
   ```

3. **Debugging tips**  
   - If the view fails to compile, run the inner SELECT alone to isolate syntax errors.  
   - Use `EXPLAIN` on the SELECT to inspect join order and potential bottlenecks.  
   - Check for NULL `orgname` by running:  
     ```sql
     SELECT COUNT(*) FROM mnaas.ra_calldate_geneva_reject WHERE orgname IS NULL;
     ```
   - Compare row counts with source tables:  
     ```sql
     SELECT COUNT(*) FROM mnaas.ra_calldate_gen_succ_rej WHERE status='REJECTED' AND row_num=1;
     SELECT COUNT(*) FROM mnaas.ra_calldate_geneva_reject;
     ```

4. **Logging** – The orchestration engine should capture the Hive/Impala console output and any error codes. Ensure the job logs are retained for at least 30 days for post‑mortem analysis.

---

## 7. External Configuration / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Location of Hive/Impala client configuration (JDBC URL, Kerberos principal, etc.). |
| `DB_CONNECTION_URL` (often set in a central `env.properties` file) | Connection string used by the job runner to reach the `mnaas` catalog. |
| `SCHEMA=mnaas` (may be injected by the orchestration layer) | Allows the same script to be reused for other schemas if needed. |
| `LOG_LEVEL` | Controls verbosity of the job runner; useful when debugging view creation failures. |

If the script references a placeholder like `${SCHEMA}` it would be substituted at runtime; however, the current file hard‑codes `mnaas`.

---

## 8. Suggested TODO / Improvements  

1. **Add a comment block** at the top of the DDL describing the purpose, owner, and last‑updated date. This aligns with the other DDL files in the repository.  
2. **Materialize the view** as a partitioned table (e.g., by `callmonth` or `processed_date`) if downstream consumption is heavy. This will improve query performance and allow incremental refreshes.  

--- 

*End of documentation.*