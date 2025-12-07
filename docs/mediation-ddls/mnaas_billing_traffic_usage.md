**File:** `mediation-ddls\mnaas_billing_traffic_usage.hql`  

---

## 1. High‑Level Summary
This script defines a Hive view `mnaas.billing_traffic_usage`. The view aggregates raw CDR‑like records from the source table `mnaas.traffic_details_billing` into a monthly‑by‑SIM usage summary. For each combination of month, TCL security ID, usage purpose, add‑on, SIM identifier, and a set of dimensional attributes (country, traffic type, call type, zone, sponsor, APN, destination type, TADIG code, filename, and the original call date), it computes total `usage`, total `actualusage`, and the count of underlying CDR rows. The view is intended for downstream billing, reporting, and reconciliation jobs.

---

## 2. Core Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_traffic_usage` | Hive **VIEW** | Provides a pre‑aggregated, month‑level usage snapshot for every SIM and related dimensions, used by downstream billing calculations. |
| `traffic_details_billing` | Hive **TABLE** (source) | Holds the raw, per‑CDR traffic details that feed the view. |

*No procedural code, functions, or classes are defined in this file; it is a pure DDL statement.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input Table** | `mnaas.traffic_details_billing` – must exist with columns: `calldate` (string/timestamp), `tcl_secs_id`, `usagefor`, `proposition_addon`, `sim`, `usage`, `actualusage`, `calltype`, `traffictype`, `country`, `zoneid`, `sponsor`, `apn`, `destinationtype`, `tadigcode`, `filename`. |
| **Output View** | `mnaas.billing_traffic_usage` – created (or replaced) by the script. |
| **Side‑Effects** | - If the view already exists, Hive will replace it (DROP + CREATE). <br>- No data is written; only metadata changes. |
| **Assumptions** | - The source table is fully populated before this view is (re)created. <br>- `calldate` is stored in a format where `substring(calldate,1,7)` yields `YYYY‑MM`. <br>- No partitioning is defined on the view; downstream jobs handle any needed partition pruning. <br>- Hive metastore is reachable and the `mnaas` database is present. |
| **External Dependencies** | - Hive / Spark‑SQL execution engine. <br>- Metastore configuration (e.g., `hive.metastore.uris`). <br>- Potential downstream jobs that reference the view (e.g., billing aggregation pipelines). |

---

## 4. Integration Points & Connectivity  

| Connected Component | How It Links |
|---------------------|--------------|
| **Other DDL scripts** (e.g., `mnaas_billing_traffic_*` views) | Those scripts often reference `billing_traffic_usage` as a source for higher‑level aggregates (e.g., usage per proposition, per sponsor). |
| **Billing ETL jobs** (Airflow/Oozie tasks) | Typical workflow: <br>1. Load raw CDRs into `traffic_details_billing`. <br>2. Run this DDL to refresh the view. <br>3. Run downstream Hive/Impala queries that join the view with pricing tables. |
| **Reporting dashboards** | BI tools query `billing_traffic_usage` directly for month‑to‑date usage metrics. |
| **Data quality checks** | Validation scripts may count rows in `traffic_details_billing` vs. `billing_traffic_usage.cdr_count` to detect missing aggregations. |

*Because the view is rebuilt each run, any job that expects a stable schema must be scheduled **after** this script completes.*

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **View recreation during high traffic** – the `CREATE VIEW … AS SELECT … GROUP BY` may cause a long-running scan of the source table, increasing Hive cluster load. | Slower downstream jobs, possible timeouts. | Schedule the DDL during off‑peak windows; consider materializing as a **managed table** with partitions if latency becomes an issue. |
| **Schema drift** – if `traffic_details_billing` gains/loses columns used in the view, the DDL will fail. | Job failure, missing billing data. | Add a pre‑execution schema validation step (e.g., `DESCRIBE` + check for required columns). |
| **Incorrect `calldate` format** – substring may produce wrong month values if the format changes. | Mis‑aligned monthly aggregates. | Enforce a canonical date format at ingestion; optionally cast to `date` and use `date_format`. |
| **Metastore connectivity loss** – view creation fails, leaving an outdated view. | Downstream jobs read stale data. | Implement retry logic in the orchestration layer; alert on Hive metastore errors. |
| **Unbounded view size** – no partitioning means full scans for any query. | Poor query performance for downstream consumers. | Consider converting the view to a partitioned **managed table** (e.g., partitioned by `callmonth`). |

---

## 6. Running / Debugging the Script  

1. **Typical Execution** (via Hive CLI, Beeline, or Spark‑SQL):  
   ```bash
   hive -f mediation-ddls/mnaas_billing_traffic_usage.hql
   # or
   beeline -u jdbc:hive2://<host>:10000/mnaas -f mediation-ddls/mnaas_billing_traffic_usage.hql
   ```

2. **Orchestration** – In Airflow, a `HiveOperator` or `SparkSubmitOperator` can point to this file; ensure `depends_on_past=False` and set `execution_timeout` appropriately.

3. **Debug Steps**  
   - **Validate source table**: `SELECT COUNT(*) FROM mnaas.traffic_details_billing LIMIT 1;`  
   - **Preview aggregation**: Run the SELECT part of the view manually with a `LIMIT 10` to verify column mapping.  
   - **Check view existence**: `SHOW CREATE VIEW mnaas.billing_traffic_usage;`  
   - **Inspect row counts**: Compare `SELECT SUM(cdr_count) FROM mnaas.billing_traffic_usage;` with `SELECT COUNT(*) FROM mnaas.traffic_details_billing;`.

4. **Log Capture** – Hive logs (`/tmp/hive.log` or the container stdout) will contain the query plan; look for “Map Reduce” stages to gauge runtime.

---

## 7. External Configuration / Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Determines Hive/Metastore connection details. |
| `HIVE_DATABASE=mnaas` (if set) | Allows the script to be run without fully‑qualified names; the script already uses fully‑qualified names, so this is optional. |
| `HIVE_EXECUTION_ENGINE` (e.g., `mr` or `tez`) | Influences performance of the aggregation. |
| **No hard‑coded file paths** – the script only references Hive objects; any external parameters (e.g., date range) are not required. |

If the organization uses a templating system (e.g., Jinja2) to inject variables, verify that the rendered file still contains the exact DDL shown above.

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Convert the view into a **partitioned managed table** (`PARTITIONED BY (callmonth STRING)`) to enable predicate push‑down and reduce scan time for downstream month‑specific queries.  
2. **Guard Against Schema Changes** – Pre‑pend a small validation block that checks for the presence of all required columns in `traffic_details_billing` and aborts with a clear error if any are missing.

---