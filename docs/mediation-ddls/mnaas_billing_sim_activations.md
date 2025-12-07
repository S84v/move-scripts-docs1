**File:** `mediation-ddls\mnaas_billing_sim_activations.hql`

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.billing_sim_activations`. The view extracts a distinct list of SIM‑related identifiers (`tcl_secs_id`, `sim`, `commercialoffer`) from the raw daily activation feed (`activations_raw_daily`) and the monthly billing reference table (`month_billing`). Only records that satisfy the production‑time window (`activationdate` between `start_with_time` and `end_with_time`), have an `ACTIVE` product status, belong to files whose name contains “HOL”, have a positive `tcl_secs_id`, and a non‑empty `commercialoffer` are retained. The view is a core input for downstream billing, usage‑aggregation, and reporting jobs.

---

## 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_sim_activations` | Hive **VIEW** | Provides a de‑duplicated, filtered set of active SIM activations with commercial offer information for the current processing window. |
| `activations_raw_daily` | Hive **TABLE** | Source of raw activation records (one row per activation event). |
| `month_billing` | Hive **TABLE** | Reference table containing billing period metadata (used here only to satisfy the FROM clause; likely provides the date window variables). |
| `start_with_time` / `end_with_time` | **HQL variables** (passed at runtime) | Define the inclusive start and end timestamps for the activation window. |
| `tcl_secs_id`, `sim`, `commercialoffer`, `productstatus`, `filename`, `activationdate` | **Columns** | Filter criteria and output fields. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Hive tables: `mnaas.activations_raw_daily`, `mnaas.month_billing`<br>- Runtime variables: `start_with_time`, `end_with_time` (expected to be in a format comparable to `activationdate` column). |
| **Outputs** | - Hive **VIEW**: `mnaas.billing_sim_activations` (contains columns `tcl_secs_id`, `sim`, `commercialoffer`). |
| **Side Effects** | - DDL operation: creates/replaces the view. If the view already exists, Hive will overwrite it (default behavior of `CREATE VIEW`). |
| **Assumptions** | - Both source tables exist and are refreshed before this script runs.<br>- Columns referenced (`tcl_secs_id`, `sim`, `commercialoffer`, `productstatus`, `filename`, `activationdate`) are present and have compatible data types.<br>- The variables `start_with_time` and `end_with_time` are supplied by the orchestration layer (e.g., Airflow, Oozie, custom scheduler).<br>- No explicit join condition is required; the script intentionally creates a Cartesian product (likely a design oversight). |
| **External Services** | - Hive/Impala metastore (execution engine).<br>- Possibly a scheduler that injects the time variables. No direct network I/O, SFTP, or API calls. |

---

## 4. Integration Points & Connectivity

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Downstream billing scripts** (e.g., `mnaas_billing_active_sims.hql`, `mnaas_billing_forked_traffic_usage.hql`) | These scripts query `billing_sim_activations` to enrich usage or revenue calculations with commercial‑offer data. |
| **Orchestration layer** (Airflow DAG, Oozie workflow, or custom batch driver) | Supplies `start_with_time` / `end_with_time` variables and schedules this DDL before any downstream SELECT‑based jobs. |
| **Data ingestion pipelines** that populate `activations_raw_daily` | Must run prior to this view creation; otherwise the view will be empty or cause errors. |
| **Metadata catalog** (e.g., Apache Atlas) | Registers the view definition for lineage tracking; downstream jobs will show a dependency on this view. |
| **Testing/validation scripts** | May run `SELECT COUNT(*) FROM mnaas.billing_sim_activations` after creation to verify row counts against expectations. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed time variables** (`start_with_time`, `end_with_time`) | View may return zero rows or cause a Hive parse error. | Validate variables in the orchestration step; abort job if they are null or not parsable. |
| **Cartesian product due to missing join condition** | Potential massive row explosion, high memory/CPU consumption, and incorrect results. | Review schema; add an explicit join (e.g., `JOIN mnaas.month_billing mb ON ard.some_key = mb.some_key`). If the join is truly unnecessary, replace the second table with a sub‑query that only provides the date window. |
| **View overwrite without versioning** | Downstream jobs may see inconsistent data if the view is recreated mid‑run. | Use `CREATE OR REPLACE VIEW` only at a well‑defined window (e.g., end of day) and ensure downstream jobs start after the view is stable. |
| **Schema drift** (column rename or type change) | Query failures in downstream jobs. | Implement schema validation step before view creation; maintain a schema registry. |
| **Permission issues** | Failure to create view if the executing user lacks `CREATE` rights on the `mnaas` database. | Ensure the service account has appropriate Hive privileges; audit periodically. |

---

## 6. Running & Debugging the Script

1. **Typical Execution (via Hive CLI):**  
   ```bash
   hive -hiveconf start_with_time=2024-11-01 -hiveconf end_with_time=2024-11-30 -f mediation-ddls/mnaas_billing_sim_activations.hql
   ```

2. **Via Scheduler (e.g., Airflow):**  
   ```python
   HiveOperator(
       task_id="create_billing_sim_activations_view",
       hql="mediation-ddls/mnaas_billing_sim_activations.hql",
       hive_cli_conn_id="hive_default",
       params={"start_with_time": "{{ ds }}", "end_with_time": "{{ next_ds }}"},
   )
   ```

3. **Debug Steps:**  
   - Verify variables: `hive -e "set;" | grep start_with_time`  
   - Run the SELECT manually: `hive -e "SELECT COUNT(*) FROM mnaas.billing_sim_activations;"`  
   - Check row distribution: `hive -e "SELECT productstatus, COUNT(*) FROM mnaas.billing_sim_activations GROUP BY productstatus;"`  
   - Inspect execution plan: `EXPLAIN SELECT * FROM mnaas.billing_sim_activations LIMIT 10;`

4. **Log Inspection:**  
   - Hive logs (`/tmp/hive.log` or the scheduler’s log) will contain the DDL statement and any errors (e.g., “SemanticException” for missing columns).

---

## 7. External Config / Environment Variables

| Variable | Source | Usage |
|----------|--------|-------|
| `start_with_time` | Orchestration (Airflow/Dag parameters, Oozie property, shell env) | Filters `activationdate` lower bound. |
| `end_with_time`   | Same as above | Filters `activationdate` upper bound. |
| Hive connection details (e.g., `hive.metastore.uris`) | Cluster configuration (Ambari, Cloudera Manager, etc.) | Required for any Hive client to connect. |
| Optional: `hive.exec.dynamic.partition` | Hive site config | Not directly used but may affect view creation if underlying tables are partitioned. |

---

## 8. Suggested Improvements (TODO)

1. **Add an explicit join condition** between `activations_raw_daily` and `month_billing` (or replace the second table with a sub‑query that only provides the date window) to avoid a Cartesian product and improve performance.
2. **Guard the DDL with `IF NOT EXISTS` or versioned naming** (e.g., `billing_sim_activations_v${run_id}`) to prevent accidental overwrites during overlapping runs and to aid lineage tracking. 

---