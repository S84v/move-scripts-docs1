**File:** `mediation-ddls\mnaas_billing_forked_traffic_usage.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view `mnaas.billing_forked_traffic_usage`. The view aggregates raw forked‑traffic CDR records (`billing_traffic_forked_cdr`) by month, subscriber, service, and a set of dimensional attributes, computing total usage, total actual usage, and the count of CDR rows. The view is a downstream data‑model component used by subsequent billing‑usage calculations (e.g., `mnaas_billing_forked_traffic_actusage.hql`, `mnaas_billing_forked_traffic_retusage.hql`) and reporting pipelines.

---

## 2. Key Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_forked_traffic_usage` | Hive **VIEW** | Provides a pre‑aggregated, month‑level snapshot of forked‑traffic usage per subscriber/service for billing and analytics. |
| `billing_traffic_forked_cdr` | Hive **TABLE** (source) | Holds raw CDR rows for forked traffic (one row per call/event). |
| `month_billing` | Hive **TABLE** (source) | Calendar dimension table mapping billing periods (`bill_month`) to a canonical `month` value. |
| `createtab_stmt` | Variable (script placeholder) | Holds the DDL string; used by the orchestration framework to execute the statement. |

*No procedural code (functions, classes) exists in this file; the script is a pure DDL definition.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Table `mnaas.billing_traffic_forked_cdr` (must contain columns: `month`, `tcl_secs_id`, `usagefor`, `proposition_addon`, `sim`, `calltype`, `traffictype`, `country`, `zoneid`, `sponsor`, `apn`, `calldate`, `destinationtype`, `tadig`, `filename`, `usage`, `actualusage`).<br>- Table `mnaas.month_billing` (must contain columns: `bill_month`, `month`). |
| **Outputs** | - Hive view `mnaas.billing_forked_traffic_usage` (persisted in the `mnaas` database). |
| **Side‑Effects** | - Overwrites the view if it already exists (implicit `CREATE OR REPLACE` semantics of Hive when the view name already exists). |
| **Assumptions** | - Hive metastore is reachable and the `mnaas` database exists.<br>- Source tables are fully populated before this script runs (typically after the CDR ingestion job).<br>- No partitioning is required on the view; downstream jobs will handle any needed filtering.<br>- The execution environment provides the `createtab_stmt` variable to the orchestration engine (e.g., Oozie, Airflow, custom scheduler). |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_billing_forked_traffic_cdr.hql`** (or similar) | Populates `billing_traffic_forked_cdr`. Must run **before** this view creation. |
| **`mnaas_month_billing.hql`** | Populates `month_billing`. Must be available prior to view creation. |
| **`mnaas_billing_forked_traffic_actusage.hql`** | Consumes `billing_forked_traffic_usage` to calculate *actual* usage per billing cycle. |
| **`mnaas_billing_forked_traffic_retusage.hql`** | Consumes the same view for *retention* usage calculations. |
| **Orchestration layer** (e.g., Oozie workflow, Airflow DAG) | Executes this script as a DDL step, typically after the CDR ingestion and month‑dimension load steps. |
| **Reporting / BI tools** (e.g., Tableau, PowerBI) | Query the view for dashboards on forked‑traffic consumption. |
| **External config** | May reference environment variables such as `HIVE_CONF_DIR`, `HADOOP_USER_NAME`, or a properties file that defines the Hive connection URL. The script itself does not embed those values; they are injected by the runner. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table schema change** (e.g., column rename) | View creation fails or produces incorrect aggregates. | Add a schema‑validation step before execution; version‑control table definitions; keep a compatibility matrix. |
| **Missing source data** (empty `billing_traffic_forked_cdr`) | View will be created but contain no rows, downstream jobs may produce zero billing. | Include a pre‑run check that row count > 0; abort with a clear error if not. |
| **Long compile time / OOM** (large CDR volume) | Job stalls, may impact scheduler resources. | Ensure Hive execution engine is tuned (e.g., Tez/LLAP); consider materializing the view as a table with periodic refresh if performance becomes an issue. |
| **View overwrite without audit** | Historical view definitions lost, making debugging harder. | Use `CREATE OR REPLACE VIEW` with a versioned naming convention (e.g., `billing_forked_traffic_usage_v${run_date}`) or keep DDL in source control and log the executed statement. |
| **Permission issues** (user cannot create view) | Job fails, downstream pipelines blocked. | Verify Hive user has `CREATE`/`ALTER` rights on the `mnaas` database; embed a permission check in the orchestration script. |

---

## 6. Running / Debugging the Script  

1. **Standard execution (via orchestration)**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_forked_traffic_usage.hql
   ```
   The orchestration tool typically injects the `createtab_stmt` variable and logs the DDL.

2. **Ad‑hoc validation**  
   ```sql
   -- Verify source tables exist
   SHOW TABLES LIKE 'billing_traffic_forked_cdr';
   SHOW TABLES LIKE 'month_billing';

   -- Preview the aggregated result (limit for quick check)
   SELECT * FROM mnaas.billing_forked_traffic_usage LIMIT 10;
   ```

3. **Debugging steps**  
   - Check Hive logs (`/tmp/hive.log` or the scheduler’s log output) for syntax errors.  
   - If the view is empty, run a count on the source tables to confirm data presence.  
   - Use `EXPLAIN` on the SELECT statement to verify join order and map‑side joins.  

4. **Re‑creation** (if you need to force a rebuild)  
   ```bash
   hive -e "DROP VIEW IF EXISTS mnaas.billing_forked_traffic_usage; \
            $(cat mediation-ddls/mnaas_billing_forked_traffic_usage.hql)"
   ```

---

## 7. External Configuration & Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_CONF_DIR` | Directory containing `hive-site.xml` for connection details. |
| `HADOOP_USER_NAME` | Hive/Hadoop user under which the script runs (must have DB permissions). |
| `ENV=prod|dev|test` (often passed by the scheduler) | Determines which Hive metastore/cluster the script targets. |
| `createtab_stmt` (internal placeholder) | Holds the DDL string; the orchestration engine may replace it with a templated version (e.g., adding `IF NOT EXISTS`). |

If any of these are missing, the job will fail at the Hive client start‑up stage.

---

## 8. Suggested Improvements (TODO)

1. **Add idempotent DDL** – Change the statement to `CREATE OR REPLACE VIEW` (or `CREATE VIEW IF NOT EXISTS`) to avoid accidental drops and to make the script safe for re‑runs.  
2. **Parameterise the view name** – Introduce a version suffix (e.g., `${run_date}`) so that historical snapshots can be retained for audit and rollback without manual intervention.

---