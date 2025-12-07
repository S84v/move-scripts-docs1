**File:** `mediation-ddls\mnaas_ra_calldate_monthly_rep.hql`

---

### 1. High‑Level Summary
This script defines a Hive/Impala view `mnaas.ra_calldate_monthly_rep`. The view aggregates monthly call‑detail‑record (CDR) statistics for four usage types – Data (DAT), SMS, A2P, and Voice‑of‑Internet (VOI) – by pulling together three logical data sets:

* **Mediation data** (accepted traffic) from `ra_calldate_traffic_table`.
* **Reject data** (rejected traffic) from `ra_calldate_rej_tbl`.
* **Generation data** (successful generation events) from `ra_calldate_gen_succ_rej`.

Each data set is filtered by the billing month supplied via the Hive variable ``month`` (populated from the `month_reports` table). The view returns per‑month totals for CDR counts and usage volumes (bytes for data/SMS/A2P, minutes for VOI), ready for downstream reporting, billing, or analytics pipelines.

---

### 2. Core Artefacts

| Artefact | Type | Responsibility |
|----------|------|-----------------|
| `createtab_stmt` | Hive variable (string) | Holds the full `CREATE VIEW … AS` statement; the orchestration engine extracts and executes it. |
| `ra_calldate_monthly_rep` | Hive view | Provides a consolidated, month‑level snapshot of mediation, reject, and generation metrics for the four usage types. |
| CTE **med_data** | Sub‑query | Normalises `ra_calldate_traffic_table` rows into separate columns per usage type (CDR count & usage). |
| CTE **med_summary** | Sub‑query | Aggregates `med_data` by `callmonth`. |
| CTE **rej_data** | Sub‑query | Pulls reject‑type rows from `ra_calldate_rej_tbl` filtered by the current billing month. |
| CTE **rej_summary** | Sub‑query | Aggregates `rej_data` by `callmonth`. |
| CTE **gen_data** | Sub‑query | Pulls successful generation rows from `ra_calldate_gen_succ_rej` filtered by the current billing month. |
| CTE **gen_summary** | Sub‑query | Aggregates `gen_data` by `callmonth`. |
| Final SELECT | Query | Joins the three summaries (LEFT OUTER) to produce a single row per month with all metrics. |

---

### 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Hive tables/views: `mnaas.ra_calldate_traffic_table`, `mnaas.ra_calldate_rej_tbl`, `mnaas.ra_calldate_gen_succ_rej`, `mnaas.month_reports`.<br>• Hive variable ``month`` (e.g., `202312`). |
| **Outputs** | • Hive view `mnaas.ra_calldate_monthly_rep` (re‑created each execution). |
| **Side‑effects** | • Overwrites the view if it already exists (DROP/CREATE semantics depend on the execution engine). |
| **Assumptions** | • All source tables exist and are populated for the target month.<br>• `usage_type` values are limited to `'DAT'`, `'SMS'`, `'A2P'`, `'VOI'`.<br>• `bytes_sec_sms` holds raw byte counts for data‑type usage; `cdr_usage` holds minutes for VOI.<br>• The orchestration layer injects the ``month`` variable correctly. |
| **External Services** | • Hive/Impala server (access credentials supplied by the job runner).<br>• Possibly a metastore service for view metadata. |

---

### 4. Integration Points

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| `mnaas_ra_calldate_med_data.hql` | Populates `ra_calldate_traffic_table` (mediation source). |
| `mnaas_ra_calldate_*_reject.hql` (e.g., `*_billing_reject`, `*_country_reject`) | Populate `ra_calldate_rej_tbl` with reject records used by this view. |
| `mnaas_ra_calldate_gen_succ_rej.hql` | Populates `ra_calldate_gen_succ_rej` (generation success/reject data). |
| `month_reports` loader script (not shown) | Supplies the mapping of logical month to `bill_month` used for filtering. |
| Down‑stream reporting jobs (e.g., monthly billing, KPI dashboards) | Query `ra_calldate_monthly_rep` for aggregated metrics. |
| Orchestration engine (Airflow, Oozie, custom scheduler) | Executes this HQL file, passing the ``month`` variable and handling the `createtab_stmt` extraction. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **View recreation during peak load** – the `CREATE VIEW` may acquire metadata locks, causing temporary query failures on dependent jobs. | Service disruption. | Schedule view refresh during off‑peak windows; use `CREATE OR REPLACE VIEW` if supported to minimise lock time. |
| **Missing or malformed ``month`` variable** – results in empty or cross‑month data. | Incorrect reporting. | Validate the variable at job start; abort with a clear error if not set. |
| **Source table schema drift** – new columns or changed data types break the CASE expressions. | Job failure. | Add schema‑validation step (e.g., `DESCRIBE` checks) before view creation. |
| **Large data volume** – full‑month scans can be expensive. | Long runtimes, resource contention. | Ensure source tables are partitioned by `callmonth`/`bill_month`; add partition pruning predicates. |
| **Incorrect unit conversion** – bytes to MB or minutes to hours may be applied inconsistently downstream. | KPI mis‑calculation. | Document conversion logic; consider storing raw units and converting only at presentation layer. |

---

### 6. Running / Debugging the Script

1. **Typical execution (via scheduler)**  
   ```bash
   hive -hiveconf month=202312 -f mediation-ddls/mnaas_ra_calldate_monthly_rep.hql
   ```
   The scheduler extracts the `createtab_stmt` variable and runs it against the Hive server.

2. **Manual test**  
   ```sql
   SET month=202312;
   -- copy the CREATE VIEW statement from the script and paste it into the Hive CLI.
   CREATE VIEW mnaas.ra_calldate_monthly_rep AS ...
   ```
   * Verify the view exists: `SHOW CREATE VIEW mnaas.ra_calldate_monthly_rep;`  
   * Spot‑check a month: `SELECT * FROM mnaas.ra_calldate_monthly_rep WHERE callmonth='202312' LIMIT 10;`

3. **Debugging tips**  
   * Run each CTE individually (replace the final SELECT with `SELECT * FROM med_data LIMIT 5;`) to confirm source data.  
   * Check row counts on source tables for the target month: `SELECT COUNT(*) FROM ra_calldate_traffic_table WHERE callmonth='202312';`  
   * Enable Hive query logging (`set hive.exec.print.summary=true;`) to see execution time per CTE.

---

### 7. External Configuration / Environment Variables

| Config Item | Purpose |
|-------------|---------|
| `HIVE_CONF_DIR` / Hive JDBC URL | Connection details for the Hive/Impala server. |
| `month` (Hive variable) | Billing month used to filter `month_reports` and source tables. |
| Optional: `HIVE_USER`, `HIVE_PASSWORD` | Authentication credentials supplied by the orchestration framework. |

If the environment uses a properties file (e.g., `hive-site.xml`), ensure the metastore and execution engine are reachable from the job host.

---

### 8. Suggested Improvements (TODO)

1. **Add Parameter Validation** – Insert a pre‑execution block that checks `:month` is non‑null and matches the `YYYYMM` pattern; abort with a clear message if invalid.
2. **Convert to a Materialized View or Incremental Table** – For large datasets, persisting the aggregated results (partitioned by `callmonth`) can dramatically reduce downstream query latency and avoid re‑scanning raw tables each month.