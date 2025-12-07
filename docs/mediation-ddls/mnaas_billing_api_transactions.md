**File:** `mediation-ddls\mnaas_billing_api_transactions.hql`

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.billing_api_transactions`. The view consolidates successful API‑subscription transactions by joining the raw API subscription feed (`api_subscription_raw`) with the monthly billing calendar (`month_billing`). It extracts a distinct list of TCL security IDs, API names, and transaction IDs for the current calendar month, filtering out records with empty API names and non‑successful statuses. Down‑stream billing, reconciliation, and reporting jobs consume this view to calculate usage‑based charges and audit API activity.

---

## 2. Core Artifact

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `createtab_stmt` | Hive DDL statement (CREATE VIEW) | Defines the logical view `mnaas.billing_api_transactions` that presents a deduplicated set of successful API transactions for the active month. |

*No procedural code, functions, or classes are present in this file; the entire file is a single DDL statement.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Source tables** | `mnaas.api_subscription_raw` (contains raw API subscription events) <br> `mnaas.month_billing` (calendar table with `partition_month` and `month` columns) |
| **Columns used** | `tcl_secs_id`, `apiname`, `transactionid` (from `api_subscription_raw`) <br> `partition_month`, `month`, `status` (from `month_billing`) |
| **Filters** | `upper(status) = 'SUCCESS'` – only successful transactions <br> `apiname != ''` – exclude empty API names |
| **Join condition** | Year and month parts of `partition_month` must match those of `month` (implicit cross‑join filtered by the two `substring` predicates). |
| **Result** | Hive **VIEW** `mnaas.billing_api_transactions` (logical, not materialized). |
| **Side effects** | Overwrites the view if it already exists. No data is written to HDFS directly. |
| **Assumptions** | • Both source tables exist and are refreshed before this script runs. <br> • `partition_month` and `month` are stored as `YYYYMM` strings (or compatible format). <br> • `status` column is present in `month_billing`. <br> • Hive metastore is reachable and the user has `CREATE VIEW` privileges. |

---

## 4. Integration Points

| Connected Component | How it Links |
|---------------------|--------------|
| **Earlier DDL scripts** (e.g., `mnaas_api_subscription_raw.hql`, `mnaas_month_billing.hql`) | Provide the underlying tables/views referenced here. |
| **Down‑stream billing aggregates** (e.g., `mnaas_billing_active_sims.hql`, `mnaas_billing_active_eids.hql`) | Likely join to `billing_api_transactions` to enrich charge calculations with API usage data. |
| **ETL pipelines** (Oozie / Airflow jobs) | Schedule this DDL after the daily load of `api_subscription_raw` and the monthly refresh of `month_billing`. |
| **Reporting dashboards** (e.g., usage audit UI) | Query the view directly for KPI calculations. |
| **Data quality monitors** | May validate that the row count of the view matches expectations (e.g., compare with source counts). |

*Because the view is defined with a *cross‑join* filtered by substring predicates, any job that expects a proper foreign‑key relationship should verify that the join does not produce Cartesian blow‑up.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table schema change** (e.g., column rename or type change) | View creation fails, downstream jobs break. | Add a schema‑validation step before executing the DDL; version‑control table definitions. |
| **Incorrect month matching** (partition format drift) | View returns data for wrong month or empty set. | Enforce a strict date format (e.g., `yyyyMM`) and add a sanity check: `SELECT COUNT(*) FROM view WHERE month = current_month`. |
| **Implicit cross‑join performance** | Large scan of both tables leading to long DDL execution or OOM. | Replace the implicit join with an explicit `JOIN` on a proper key (e.g., `ON month_billing.month = api_subscription_raw.month`). |
| **View overwrites without backup** | Accidental loss of previous view definition. | Use `CREATE OR REPLACE VIEW` with versioned naming (e.g., `billing_api_transactions_v20251204`). |
| **Missing privileges** | DDL fails, pipeline stalls. | Include a pre‑flight check for `CREATE VIEW` rights; document required Hive role. |

---

## 6. Running & Debugging the Script

1. **Typical execution (batch)**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_api_transactions.hql
   ```
   *Often wrapped in an Oozie workflow or Airflow DAG that runs after the daily raw load.*

2. **Ad‑hoc verification**  
   ```sql
   -- After view creation
   SELECT COUNT(*) AS cnt,
          COUNT(DISTINCT tcl_secs_id) AS uniq_tcl,
          MIN(month) AS min_month,
          MAX(month) AS max_month
   FROM mnaas.billing_api_transactions;
   ```
   *Check that `cnt` is non‑zero and that `min_month`/`max_month` align with the current processing month.*

3. **Debugging failures**  
   - **Error: Table not found** – Verify that `api_subscription_raw` and `month_billing` exist (`SHOW TABLES LIKE 'api_subscription_raw';`).  
   - **Error: Column not found** – Confirm column names (`DESCRIBE mnaas.api_subscription_raw;`).  
   - **Performance slowness** – Run an `EXPLAIN` on the SELECT part to see the join plan; consider adding partition filters or rewriting the join.

4. **Log capture**  
   - In Oozie, enable `hive.log.dir` to capture the Hive console output.  
   - In Airflow, set `log_level='INFO'` for the HiveOperator.

---

## 7. External Configuration / Environment Variables

The script itself does not reference any external config files or environment variables. However, typical deployment environments inject:

| Variable | Purpose |
|----------|---------|
| `HIVE_CONF_DIR` | Location of Hive configuration (metastore URI, authentication). |
| `HADOOP_USER_NAME` | Hive execution user (must have `CREATE VIEW` rights). |
| `PROCESSING_DATE` (optional) | If the pipeline uses a date parameter, the underlying tables may be partitioned by this value; the view relies on the `partition_month` column matching the calendar month. |

If the environment uses a templating engine (e.g., Jinja) to substitute values, verify that the rendered script still contains the static DDL shown above.

---

## 8. Suggested Improvements (TODO)

1. **Replace implicit cross‑join with an explicit join**  
   ```sql
   CREATE OR REPLACE VIEW mnaas.billing_api_transactions AS
   SELECT DISTINCT a.tcl_secs_id,
                   a.apiname,
                   a.transactionid
   FROM   mnaas.api_subscription_raw a
   JOIN   mnaas.month_billing b
     ON   substr(b.partition_month,1,4) = substr(a.month,1,4)
    AND   substr(b.partition_month,5,6) = substr(a.month,6,7)
   WHERE  upper(b.status) = 'SUCCESS'
     AND  a.apiname <> '';
   ```
   This removes the Cartesian product risk and improves planner optimization.

2. **Add a comment block with lineage metadata**  
   ```sql
   /* --------------------------------------------------------------
    * View: billing_api_transactions
    * Source: api_subscription_raw (raw API events)
    *         month_billing (calendar & status)
    * Owner:   Data Engineering – Billing
    * Created: 2025-12-04
    * Purpose: Provide distinct successful API transactions per month.
    * -------------------------------------------------------------- */
   ```

These changes increase maintainability, performance, and traceability.