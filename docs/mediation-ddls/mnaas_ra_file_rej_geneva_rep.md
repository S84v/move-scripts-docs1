**File:** `mediation-ddls\mnaas_ra_file_rej_geneva_rep.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.ra_file_rej_geneva_rep`. The view consolidates rejection statistics for Geneva‑originated files, pivoting the raw rejection table `mnaas.ra_file_rej_geneva` so that each file/month has separate columns for Data (DAT), Voice (VOI) and SMS (SMS) CDR counts and usage totals. The view is used downstream by reporting and analytics jobs that need a flat, month‑by‑file view of rejection metrics.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_file_rej_geneva` | Hive table (source) | Holds raw rejection rows: `filename`, `month`, `usage_type` (`DAT`/`VOI`/`SMS`), `cdr_usage`, `cdr_count`. |
| `gen_rej` | CTE | Simple projection of the source table (keeps all columns). |
| `dat_rej` | CTE | Filters `gen_rej` for `usage_type='DAT'` and renames usage/count columns to `dat_gen_rej_usg`, `dat_gen_rej_cdr`. |
| `voi_rej` | CTE | Same as above for `usage_type='VOI'`. |
| `sms_rej` | CTE | Same as above for `usage_type='SMS'`. |
| `ra_file_rej_geneva_rep` | Hive **VIEW** | Joins the three CTEs on `filename` & `month` (full outer) and coalesces nulls to zero, producing one row per file/month with six metric columns (count & usage for each usage type). |

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Input Table** | `mnaas.ra_file_rej_geneva` – must exist with columns `filename` (string), `month` (string/int), `usage_type` (string), `cdr_usage` (numeric), `cdr_count` (int). |
| **Output Object** | Hive **VIEW** `mnaas.ra_file_rej_geneva_rep`. No physical data is stored; the view is recomputed on each query. |
| **Side‑Effects** | DDL operation – creates (or overwrites) the view in the `mnaas` database. May replace an existing view with the same name. |
| **Assumptions** | • Hive/Impala engine is available and the user executing the script has `CREATE VIEW` privilege on `mnaas`. <br>• The source table’s schema is stable (no column rename or type change). <br>• No partitioning is required for this view (it is a logical aggregation). |
| **External Services** | Hive metastore, underlying HDFS storage for the source table. No network calls, SFTP, or external APIs are invoked. |

---

## 4. Integration Points  

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| `mnaas_ra_file_rej_geneva.hql` (previous in history) | Creates/populates the source table `ra_file_rej_geneva`. This view must be built **after** that script runs. |
| Reporting jobs (e.g., daily KPI aggregation, billing reconciliation) | Query `ra_file_rej_geneva_rep` to obtain per‑file/month rejection metrics. |
| ETL orchestration (Oozie / Airflow / custom scheduler) | Typically a step in a DAG: <br>1️⃣ Load raw rejection data → `ra_file_rej_geneva` (via `mnaas_ra_file_rej_geneva.hql`). <br>2️⃣ Build view → this script. <br>3️⃣ Consume view in downstream analytics. |
| Data catalog / lineage tools | Will register the view as a derived dataset, linking back to the source table. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table missing or schema drift** | View creation fails; downstream jobs break. | Add a pre‑check (`SHOW TABLES LIKE 'ra_file_rej_geneva'`) and validate column list before `CREATE VIEW`. |
| **Name collision – existing view with incompatible definition** | Overwrites silently, possibly breaking consumers. | Use `CREATE OR REPLACE VIEW` with explicit versioning (e.g., `ra_file_rej_geneva_rep_v20251204`). |
| **Null handling errors** (e.g., `isnull` vs `nvl` depending on engine) | Incorrect metric values (null instead of 0). | Verify engine compatibility; consider using `COALESCE` for portability. |
| **Performance degradation on large source tables** | Full outer joins may be expensive. | Ensure source table is partitioned by `month` and/or `filename`; consider materialized view or incremental table if needed. |
| **Insufficient privileges** | DDL fails, pipeline stops. | Grant `CREATE VIEW` on `mnaas` to the service account; audit permissions regularly. |

---

## 6. Execution & Debugging Guide  

1. **Run the script** (Hive CLI or Beeline):  
   ```bash
   hive -f mediation-ddls/mnaas_ra_file_rej_geneva_rep.hql
   # or
   beeline -u jdbc:hive2://<host>:10000/mnaas -f mediation-ddls/mnaas_ra_file_rej_geneva_rep.hql
   ```

2. **Validate creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.ra_file_rej_geneva_rep;
   ```

3. **Sample query to verify data**:  
   ```sql
   SELECT * FROM mnaas.ra_file_rej_geneva_rep
   ORDER BY month DESC, filename
   LIMIT 20;
   ```

4. **Debugging tips**:  
   * If the view fails to compile, run the inner CTEs individually (`SELECT * FROM gen_rej LIMIT 5;`) to isolate schema issues.  
   * Use `EXPLAIN` on the final SELECT to see join strategy and spot potential bottlenecks.  
   * Check Hive logs (`/var/log/hive/hive.log`) for syntax errors related to `isnull`/`nvl` functions.  

---

## 7. External Configuration / Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Determines Hive metastore connection; required for any Hive DDL. |
| `DATABASE` (if templated) | In some pipelines the database name (`mnaas`) may be injected via a variable; ensure the variable resolves correctly. |
| `HIVE_EXECUTION_ENGINE` | Should be set to `mr` or `tez` depending on cluster; affects performance of the full outer joins. |

If the script is rendered from a template engine (e.g., Jinja), verify that placeholders for database or table names are correctly substituted before execution.

---

## 8. Suggested Improvements (TODO)

1. **Add idempotent creation** – replace the raw `CREATE VIEW` with `CREATE OR REPLACE VIEW` (or wrap in a `DROP VIEW IF EXISTS` block) to make the step safe for re‑runs.  
2. **Document engine‑specific functions** – replace `isnull`/`nvl` with the standard `COALESCE` function to improve portability across Hive, Spark SQL, and Presto.  

---