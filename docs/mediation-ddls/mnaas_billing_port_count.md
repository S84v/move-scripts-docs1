**File:** `mediation-ddls\mnaas_billing_port_count.hql`

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.billing_port_count`. The view extracts a normalized “TC‑LSEC” identifier (`tcl_secs_id`) from the raw MNP port‑in/out details, joins it with the monthly billing calendar, and returns the distinct combination of that identifier, product ID, and port type for all records whose `portstartdate` falls within the current billing month window (`start_with_time` – `end_with_time`). The view is used downstream by billing aggregation jobs to count ports per product and port type.

---

## 2. Core Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_port_count` | Hive **VIEW** | Provides a de‑duplicated list of `(tcl_secs_id, productid, porttype)` for ports active in the current billing month. |
| `base_recs` (CTE) | **WITH** clause | Normalises `customernumber` to a numeric `tcl_secs_id` and filters raw port records to the current month window. |
| `mnaas.mnp_portinout_details_raw` | Source **TABLE** | Holds raw MNP port‑in/out events (fields: `customernumber`, `productid`, `porttype`, `portstartdate`, …). |
| `mnaas.month_billing` | Source **TABLE** | Holds the billing calendar with columns `start_with_time` and `end_with_time` defining the active month. |

*No procedural code, functions, or classes are defined in this file; it is a pure DDL statement.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.mnp_portinout_details_raw` (raw port data) <br> - `mnaas.month_billing` (billing calendar) |
| **Outputs** | - Hive **VIEW** `mnaas.billing_port_count` (non‑materialised; recomputed on each query) |
| **Side‑Effects** | - Overwrites the view if it already exists (Hive `CREATE VIEW` will fail if the view exists; in practice the script is usually run with `CREATE OR REPLACE VIEW` via a wrapper). |
| **Assumptions** | - `customernumber` is either the literal `'311234567'` (mapped to `25050`) or follows the pattern `<prefix>_<numeric>` where the numeric part can be cast to `INT`. <br> - `portstartdate`, `start_with_time`, `end_with_time` are comparable timestamp/date types. <br> - Source tables are present and populated before this script runs. <br> - Hive metastore is reachable and the user has `CREATE VIEW` privileges on schema `mnaas`. |
| **External Services** | - Hive/Tez/LLAP execution engine (configured via environment variables or Hive client config). <br> - Potential downstream jobs (e.g., billing aggregation Spark jobs) that query this view. |

---

## 4. Integration Points & Call Graph

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| **`mnaas_billing_*`** view scripts (e.g., `mnaas_billing_active_sims.hql`) | May join to `billing_port_count` to enrich port‑count metrics with SIM or EID data. |
| **ETL orchestration (Oozie / Airflow) DAG** | Executes this DDL as a prerequisite step before any billing usage aggregation jobs that rely on the view. |
| **Reporting / BI layer** (e.g., Tableau, PowerBI) | Queries `billing_port_count` directly or via downstream materialised tables for dashboards on port activity per product. |
| **Data quality checks** (e.g., `mnaas_billing_port_count_check.hql` if exists) | Validate that the view returns expected row counts; may be scheduled after view creation. |
| **Configuration files** (`hive-site.xml`, environment vars like `HIVE_CONF_DIR`) | Provide connection details, execution engine settings, and any required Kerberos tickets. |

*Note:* The view does **not** write data; it is read‑only for downstream consumers.

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table schema change** (e.g., `customernumber` renamed) | View creation fails or returns wrong data. | Add schema‑validation step in the orchestration before running the DDL; version‑control table definitions. |
| **Unexpected `customernumber` format** (missing `_` or non‑numeric suffix) | `CAST(... AS INT)` throws runtime error, causing the view creation to abort. | Extend the `decode` logic with a safe fallback (`COALESCE`) or add a `TRY_CAST` (if Hive version supports) and log malformed rows. |
| **Missing billing calendar row for the current month** | `base_recs` returns zero rows → downstream jobs may produce empty results. | Ensure month‑billing table is pre‑populated; add a health‑check that verifies a row exists for the target month. |
| **Performance degradation on large raw tables** | Full scan of `mnp_portinout_details_raw` each query; may cause long runtimes. | Partition `mnp_portinout_details_raw` by `portstartdate` (or month) and add a predicate push‑down; consider materialising the view as a table with daily partitions. |
| **View name collision** (existing view with different definition) | Inconsistent data if the view is not refreshed. | Use `CREATE OR REPLACE VIEW` (or drop‑and‑create) in the deployment script; enforce naming conventions. |

---

## 6. Running & Debugging the Script

1. **Typical execution (via Hive CLI / Beeline)**  
   ```bash
   beeline -u "jdbc:hive2://<hive-host>:10000/mnaas" -n <user> -p <password> -f mediation-ddls/mnaas_billing_port_count.hql
   ```

2. **Within an Oozie workflow**  
   ```xml
   <action name="create_billing_port_count_view">
       <hive xmlns="uri:oozie:hive-action:0.5">
           <job-tracker>${jobTracker}</job-tracker>
           <name-node>${nameNode}</name-node>
           <script>mediation-ddls/mnaas_billing_port_count.hql</script>
           <param>--hiveconf hive.execution.engine=tez</param>
       </hive>
       <ok to="next-step"/>
       <error to="fail"/>
   </action>
   ```

3. **Debugging steps**  
   - **Validate source tables**: `SHOW CREATE TABLE mnaas.mnp_portinout_details_raw;` and `SELECT COUNT(*) FROM mnaas.month_billing WHERE start_with_time <= CURRENT_DATE AND end_with_time >= CURRENT_DATE;`.  
   - **Run the CTE alone** to inspect intermediate rows:  
     ```sql
     WITH base_recs AS ( ... ) SELECT * FROM base_recs LIMIT 10;
     ```  
   - **Check view definition** after creation: `SHOW CREATE VIEW mnaas.billing_port_count;`.  
   - **Inspect logs**: HiveServer2 logs (`/var/log/hive/hiveserver2.log`) for any CAST/DECODE errors.

---

## 7. External Configuration & Environment Variables

| Config / Variable | Purpose |
|-------------------|---------|
| `HIVE_CONF_DIR` / `hive-site.xml` | Hive connection parameters, execution engine, metastore URI. |
| `HADOOP_USER_NAME` (or Kerberos principal) | Authentication for Hive/HCatalog. |
| Optional: `BILLING_MONTH` (if the script is templated) | Could be used to filter `month_billing` to a specific month; not present in the current static script but may be injected by the orchestration layer. |

*If the script is invoked through a wrapper (e.g., a Bash or Python driver), those wrappers may pass additional `--hiveconf` parameters.*

---

## 8. Suggested Improvements (TODO)

1. **Add defensive parsing for `customernumber`**  
   Replace the raw `CAST(split_part(... ) AS INT)` with `TRY_CAST` (Hive 3+) or a `CASE` that defaults to a sentinel value and logs the offending rows.

2. **Materialise the view for performance**  
   Create a partitioned table (e.g., `billing_port_count_daily`) refreshed nightly, then replace the view with a simple `SELECT * FROM billing_port_count_daily`. This reduces the cost of scanning the raw port table on every downstream query.

---