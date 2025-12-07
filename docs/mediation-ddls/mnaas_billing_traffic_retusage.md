**File:** `mediation-ddls\mnaas_billing_traffic_retusage.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.billing_traffic_retusage`. The view aggregates raw CDR‑level records from `mnaas.traffic_details_billing` into monthly‑by‑SIM usage statistics, separating inbound/outbound traffic, domestic vs. international/roaming, and counting CDRs. The view is a downstream data source for billing‑related jobs such as `mnaas_billing_traffic_actusage`, `mnaas_billing_traffic_filtered_cdr`, and any downstream revenue‑recognition or reporting processes.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| **`billing_traffic_retusage`** | Hive/Impala **VIEW** | Provides a pre‑aggregated, month‑level snapshot of traffic usage per SIM (and related dimensions) for downstream billing calculations. |
| **`createtab_stmt`** (implicit) | Variable holding the DDL string | Used by the deployment framework to execute the CREATE VIEW statement. |

*No procedural code (UDFs, stored procedures) is defined in this file.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input Table** | `mnaas.traffic_details_billing` – raw CDR data with columns: `calldate`, `tcl_secs_id`, `usagefor`, `proposition_addon`, `sim`, `calltype`, `traffictype`, `country`, `zoneid`, `sponsor`, `apn`, `destinationtype`, `tadigcode`, `filename`, `usage` (numeric). |
| **Output View** | `mnaas.billing_traffic_retusage` – columns: <br>`callmonth` (YYYY‑MM), `tcl_secs_id`, `usagefor`, `proposition_addon`, `sim`, `calltype`, `traffictype`, `country`, `zoneid`, `sponsor`, `apn`, `calldate`, `destinationtype`, `tadigcode`, `filename`, <br>`usage` (SUM), `cdr_count` (COUNT), `in_usage`, `in_cdr`, `out_usage`, `out_cdr`, `in_dom_usage`, `in_dom_cdr`, `in_int_rom_usage`, `in_int_rom_cdr`, `out_dom_usage`, `out_dom_cdr`, `out_int_rom_usage`, `out_int_rom_cdr`, `rom_count`. |
| **Side Effects** | - Creates or replaces the view (DROP/CREATE semantics depend on the execution engine). <br>- No data mutation; only metadata change. |
| **Assumptions** | - `traffic_details_billing` exists and is populated before this script runs. <br>- `calldate` is a string (or timestamp) where `substring(calldate,1,7)` yields `YYYY‑MM`. <br>- All referenced columns are non‑null or handled implicitly by `SUM`/`COUNT`. <br>- The execution environment has privileges to create views in the `mnaas` schema. |

---

## 4. Integration Points

| Connected Component | How It Links |
|---------------------|--------------|
| **`mnaas_billing_traffic_actusage.hql`** | Likely consumes `billing_traffic_retusage` to compute actual usage charges. |
| **`mnaas_billing_traffic_filtered_cdr.hql`** | May filter raw CDRs before they land in `traffic_details_billing`; this view aggregates the filtered result. |
| **`mnaas_billing_traffic_forked_cdr.hql`** | May split CDRs into separate streams; the view provides a unified aggregation for billing. |
| **ETL orchestration scripts** (e.g., Airflow DAGs, Oozie workflows) | Call this DDL as a “DDL step” after the raw CDR load and before downstream billing jobs. |
| **Reporting / BI tools** | Query the view directly for month‑by‑SIM usage dashboards. |
| **Configuration** | The deployment framework may inject the target database name (`mnaas`) via an environment variable or a properties file; the script itself does not reference external config. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift in `traffic_details_billing`** (new/renamed columns) | View creation fails or returns wrong results. | Include a schema‑validation step before view creation; version‑control the source table DDL. |
| **Performance degradation** (large GROUP BY on many columns) | Long view refresh times, downstream job delays. | Ensure `traffic_details_billing` is partitioned (e.g., by `callmonth` or `calldate`), and that relevant columns are bucketed or indexed. Consider materialized view or incremental aggregation if supported. |
| **Incorrect `calldate` format** | `callmonth` extraction yields malformed values, breaking downstream month‑based logic. | Validate `calldate` format during the upstream CDR ingestion; add a defensive `CASE WHEN` to default to NULL or a sentinel month. |
| **Missing privileges** | View creation aborts, pipeline stops. | Verify that the service account used by the orchestration engine has `CREATE VIEW` rights on schema `mnaas`. |
| **Data quality – nulls in key columns** | Aggregations may group unexpected rows, inflating counts. | Add `COALESCE` defaults for key columns in the SELECT if nulls are possible, or enforce NOT NULL constraints upstream. |

---

## 6. Execution & Debugging Guide

1. **Run the script**  
   - Typically invoked by the orchestration engine (e.g., `hive -f mnaas_billing_traffic_retusage.hql` or via a Spark‑SQL `spark.sql(createtab_stmt)` call).  
   - Ensure the environment points to the correct Hive/Impala server and the `mnaas` database is selected.

2. **Validate creation**  
   ```sql
   SHOW CREATE VIEW mnaas.billing_traffic_retusage;
   DESCRIBE FORMATTED mnaas.billing_traffic_retusage;
   ```

3. **Sample data check**  
   ```sql
   SELECT callmonth, sim, usage, cdr_count
   FROM mnaas.billing_traffic_retusage
   WHERE callmonth = '2024-09'
   LIMIT 10;
   ```

4. **Performance check**  
   - Run `EXPLAIN` on a representative query against the view to verify that partition pruning is applied.  
   - Monitor query duration via the Hive/Impala UI or Spark UI.

5. **Debugging failures**  
   - If the view fails to create, inspect the Hive/Impala logs for “Table not found” or “Permission denied”.  
   - Verify that `traffic_details_billing` is accessible and that all column names match exactly.  
   - Use `SELECT * FROM mnaas.traffic_details_billing LIMIT 1;` to confirm column existence.

---

## 7. External Config / Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| **`DB_NAME`** (optional) | Some deployment wrappers replace the hard‑coded `mnaas` schema with a variable; not referenced directly in this file. |
| **Hive/Impala connection properties** (e.g., `hive.metastore.uris`, `spark.sql.warehouse.dir`) | Provided by the orchestration environment; required for the DDL to execute. |
| **`createtab_stmt`** container | In many pipelines the DDL is stored in a variable (`createtab_stmt`) that the runner executes; the variable name is implied by the surrounding framework. |

If the production environment uses a templating engine (e.g., Jinja, Velocity), the schema name may be substituted at runtime.

---

## 8. Suggested Improvements (TODO)

1. **Add explicit partitioning** – Convert the view into a **partitioned table** (e.g., `PARTITIONED BY (callmonth STRING)`) to improve query performance for month‑specific downstream jobs.  
2. **Guard against malformed dates** – Wrap the `substring(calldate,1,7)` expression in a `CASE` that returns a default month (`'1900-01'`) when `calldate` does not match the expected pattern, preventing view creation failures.

---