**File:** `mediation-ddls\mnaas_billing_sim_ipvprobe.hql`  

---

## 1. High‑Level Summary
This script creates the Hive view `mnaas.billing_sim_ipvprobe`. The view extracts SIM‑related IPv probe records (`secsid`, `iccid`, `partition_date`) from the raw daily probe table (`mnaas.ipvprobe_raw_daily`) and limits the rows to the billing month defined in `mnaas.month_billing` by applying a `partition_date BETWEEN start_date AND end_date` filter. Down‑stream billing and mediation jobs (e.g., usage aggregation, SIM activation, product activation) reference this view to correlate IPv probe data with the current billing cycle.

---

## 2. Core Artifact

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `createtab_stmt` | Hive DDL statement | Defines the view `mnaas.billing_sim_ipvprobe` that joins raw IPv probe data with the month‑billing window. |

*No procedural code, functions, or classes are present in this file; the entire file is a single DDL statement.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs (tables/columns)** | - `mnaas.ipvprobe_raw_daily` (expects columns `secsid`, `iccid`, `partition_date`)<br>- `mnaas.month_billing` (expects columns `start_date`, `end_date`) |
| **Outputs** | Hive **view** `mnaas.billing_sim_ipvprobe` (no physical data written; query results are computed at runtime). |
| **Side Effects** | - Registers the view in the Hive metastore.<br>- Overwrites an existing view with the same name (Hive `CREATE VIEW` will fail if the view exists unless `OR REPLACE` is used; this script does not include `OR REPLACE`). |
| **Assumptions** | - Both source tables are present and up‑to‑date when the script runs.<br>- `partition_date` is a date‑compatible column that aligns with `start_date`/`end_date` from `month_billing`.<br>- No explicit Hive configuration (e.g., `hive.exec.dynamic.partition`) is required for this view. |
| **External Services** | Hive/Impala metastore, underlying HDFS storage for the source tables. No network calls, SFTP, or external APIs. |

---

## 4. Integration Points  

| Connected Component | How the View Is Used |
|---------------------|----------------------|
| **Down‑stream billing jobs** (e.g., `mnaas_billing_active_sims.hql`, `mnaas_billing_forked_traffic_usage.hql`) | Join `billing_sim_ipvprobe` to enrich usage records with SIM identifiers and billing‑period filtering. |
| **Orchestration layer** (e.g., Oozie, Airflow, custom scheduler) | Typically invoked after the `month_billing` table is refreshed for the new billing cycle and before any usage aggregation jobs that need IPv probe data. |
| **Data quality / monitoring scripts** | May query the view to validate that the number of rows matches expectations for the current month. |
| **Reporting / analytics** | BI tools (e.g., Tableau, PowerBI) can reference the view for ad‑hoc analysis of IPv probe activity per SIM. |

*Note:* The view name follows the naming convention used across the `mediation-ddls` package (`billing_<entity>`), making it discoverable by any script that dynamically builds its SELECT list based on the `information_schema.views` metadata.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **View creation fails because the view already exists** | Job aborts; downstream jobs that depend on the view cannot run. | Use `CREATE OR REPLACE VIEW` or add a pre‑check (`DROP VIEW IF EXISTS`) in the script. |
| **Source tables missing or schema changed** | Empty or malformed view; downstream jobs produce incorrect billing. | Add a pre‑execution validation step that checks table existence and required columns (e.g., `DESCRIBE FORMATTED`). |
| **Large join without partition pruning** (if `month_billing` contains many rows) | Excessive query time, possible OOM on Hive server. | Ensure `month_billing` is a small dimension table (one row per month) and that `partition_date` is partitioned in `ipvprobe_raw_daily`. |
| **Incorrect date range due to time‑zone or data‑type mismatch** | Records from wrong billing cycle are included/excluded. | Verify that `partition_date`, `start_date`, `end_date` are stored as the same data type (e.g., `DATE`) and that time‑zone handling is consistent. |
| **Missing refresh of `month_billing` before view creation** | View may use an outdated billing window. | Enforce ordering in the orchestration DAG: `month_billing` refresh → view creation → usage jobs. |

---

## 6. Running / Debugging the Script  

1. **Standard execution** (via Hive CLI or Beeline):  
   ```bash
   hive -f mediation-ddls/mnaas_billing_sim_ipvprobe.hql
   # or
   beeline -u jdbc:hive2://<host>:10000 -f mediation-ddls/mnaas_billing_sim_ipvprobe.hql
   ```

2. **Verify creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.billing_sim_ipvprobe;
   SELECT COUNT(*) FROM mnaas.billing_sim_ipvprobe LIMIT 10;
   ```

3. **Debug steps if the view is not created**:  
   - Check Hive logs for “already exists” errors.  
   - Run `DESCRIBE FORMATTED mnaas.ipvprobe_raw_daily;` and `DESCRIBE FORMATTED mnaas.month_billing;` to confirm required columns.  
   - Execute the SELECT part alone to validate row counts and date filtering:  
     ```sql
     SELECT secsid, iccid, partition_date
     FROM mnaas.ipvprobe_raw_daily, mnaas.month_billing
     WHERE partition_date BETWEEN start_date AND end_date
     LIMIT 10;
     ```

4. **In an orchestrated environment** (e.g., Airflow DAG):  
   - Ensure the task’s `depends_on_past` or `upstream` includes the `month_billing` refresh task.  
   - Use the HiveOperator with `hive_cli_conn_id` and set `hiveconf` if any runtime variables are needed.

---

## 7. External Configuration / Environment Variables  

| Config Item | Usage |
|-------------|-------|
| Hive metastore connection (e.g., `hive.metastore.uris`) | Required for any Hive DDL execution; defined outside this script (usually in `hive-site.xml`). |
| Optional: `HIVE_CONF` environment variables | May be used by the orchestration wrapper to set session parameters (e.g., `hive.exec.dynamic.partition.mode`). Not referenced directly in the script. |
| No file‑level placeholders (e.g., `${START_DATE}`) are present; the script relies on column values from `month_billing`. |

If the deployment environment injects variables (e.g., via Oozie `workflow.xml`), they would need to be added to the script; currently none are required.

---

## 8. Suggested Improvements (TODO)

1. **Make the view creation idempotent** – prepend the DDL with `DROP VIEW IF EXISTS mnaas.billing_sim_ipvprobe;` or switch to `CREATE OR REPLACE VIEW`. This prevents failures on re‑runs and simplifies orchestration retries.

2. **Add explicit column comments** – enrich the view definition with `COMMENT` clauses for each column (e.g., `secsid COMMENT 'Secure SIM identifier'`). This improves data‑dictionary generation and downstream developer understanding.  

---