**File:** `mediation-ddls\mnaas_mnp_porters_report.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.mnp_porters_report`. The view aggregates Mobile Number Portability (MNP) port‑in/out activity per month, per TCL (telecom carrier) sector ID, and per organization. It derives the TCL sector ID from the `customernumber` field (special‑case handling for a test MSISDN), filters records to the current month‑report window, joins to the organization master table, and returns the distinct count of products (i.e., subscriber count) for each combination of month, TCL sector, organization name, and port type.

---

## 2. Important Objects & Responsibilities

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.mnp_porters_report` | **Hive View** | Exposes a ready‑to‑consume aggregation of MNP port‑in/out activity for downstream reporting / billing processes. |
| `base_recs` (CTE) | **Derived Table** | Normalises raw port‑in/out rows, maps `customernumber` → `tcl_secs_id`, and restricts rows to the active month window (`portstartdate` between `start_with_time` and `end_with_time`). |
| `mnaas.mnp_portinout_details_raw` | **Source Table** | Holds raw MNP port‑in/out event records (fields: `customernumber`, `portstartdate`, `productid`, `porttype`, …). |
| `mnaas.month_reports` | **Reference Table** | Provides the month‑level time window (`start_with_time`, `end_with_time`, `month`). |
| `mnaas.org_details` | **Lookup Table** | Maps TCL sector IDs (`orgno`) to human‑readable organization names (`orgname`). |
| `decode(customernumber, '311234567', 25050, CAST(split_part(customernumber, '_', 2) AS INT))` | **Expression** | Translates a special test MSISDN (`311234567`) to a fixed sector ID (25050); otherwise extracts the numeric sector ID from the second token of `customernumber` (delimited by `_`). |

---

## 3. Data Flow – Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.mnp_portinout_details_raw` (raw MNP events) <br> - `mnaas.month_reports` (month boundaries) <br> - `mnaas.org_details` (org‑lookup) |
| **Output** | Hive **view** `mnaas.mnp_porters_report` with columns: <br> `month` (STRING/DATE) <br> `tcl_secs_id` (INT) <br> `orgname` (STRING) <br> `porttype` (STRING) <br> `subs_count` (BIGINT) |
| **Side Effects** | - Creates or replaces the view definition in the `mnaas` database. <br> - No data mutation; only metadata change. |
| **Assumptions** | - All referenced tables exist and are refreshed before this script runs. <br> - `customernumber` follows the pattern `<prefix>_<tcl_secs_id>` except for the test value `311234567`. <br> - `portstartdate`, `start_with_time`, `end_with_time` are comparable timestamp/date types. <br> - `productid` uniquely identifies a subscriber/line. <br> - Hive/Impala engine supports `decode` and `split_part` functions (standard Hive UDFs). |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnp_portinout_details_raw` creation scripts** (e.g., `mnaas_mnp_portinout_details_raw.hql`) | Provide the raw event data that this view consumes. |
| **`month_reports` generation script** (e.g., `mnaas_month_reports.hql`) | Supplies the month window used to filter records. |
| **`org_details` master data load** (e.g., `mnaas_org_details.hql`) | Supplies organization names; any change here will affect the view’s `orgname` column. |
| **Downstream reporting jobs** (e.g., daily/weekly MNP KPI dashboards, billing reconciliation jobs) | Query `mnaas.mnp_porters_report` directly; they rely on its schema and semantics. |
| **Orchestration layer** (Oozie / Airflow DAG) | Typically runs this DDL as a “schema preparation” step before the reporting jobs. |
| **CI/CD pipeline** | May include a validation step that runs `SHOW CREATE VIEW mnaas.mnp_porters_report` and compares it against a stored baseline. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – underlying tables add/rename columns or change data types. | View creation fails or returns wrong results. | Add a pre‑deployment validation step that runs `DESCRIBE` on each source table and aborts if expected columns are missing. |
| **Incorrect `customernumber` parsing** – new formats break the `split_part` logic. | Wrong `tcl_secs_id` → mis‑attributed counts. | Centralise the parsing logic into a UDF (`parse_tcl_secs_id`) and version‑control it; add unit tests for known patterns. |
| **Performance bottleneck** – `COUNT(DISTINCT productid)` on large raw table without partition pruning. | Long job runtimes, possible OOM. | Ensure `mnp_portinout_details_raw` is partitioned by `portstartdate` (or month) and that the `WHERE` clause pushes the partition filter. Consider using `approx_count_distinct` if exactness is not required. |
| **View stale after source table refresh** – if raw tables are recreated (DROP/CREATE) the view may become invalid. | Query failures. | Use `CREATE OR REPLACE VIEW` (already implied) and schedule view recreation after any source table rebuild. |
| **Hard‑coded test MSISDN** – future test numbers may be added. | Unexpected sector IDs. | Move the mapping to a small reference table (`test_mnp_mapping`) and join instead of hard‑coding. |

---

## 6. Running / Debugging the Script

1. **Typical Execution** (via Hive CLI or embedded in an Oozie action):  
   ```bash
   hive -f mediation-ddls/mnaas_mnp_porters_report.hql
   ```
   - The script will either create the view (if absent) or replace it.

2. **Verification Steps**  
   - After execution, run:  
     ```sql
     SHOW CREATE VIEW mnaas.mnp_porters_report;
     SELECT * FROM mnaas.mnp_porters_report LIMIT 10;
     ```
   - Validate that `month`, `tcl_secs_id`, `orgname`, `porttype`, and `subs_count` appear as expected.

3. **Debugging Common Issues**  
   - **Error: Table not found** – Verify that `mnp_portinout_details_raw`, `month_reports`, and `org_details` are present in the `mnaas` database.  
   - **Error: Invalid function** – Ensure Hive version supports `decode` and `split_part`; otherwise replace with `CASE WHEN` and `regexp_extract`.  
   - **Unexpected counts** – Run the CTE (`base_recs`) alone to inspect intermediate rows and confirm the date filter and `tcl_secs_id` mapping.

4. **Logging**  
   - In an Oozie workflow, capture the Hive command’s stdout/stderr to a log file (`${wf:logDir}/mnp_porters_report.log`) for post‑run audit.

---

## 7. External Configuration / Environment Variables

| Config Item | Usage |
|-------------|-------|
| `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Determines Hive/Metastore connection; not referenced directly in the script but required for any Hive execution environment. |
| `DB_NAME` (if templated) | The script hard‑codes the database `mnaas`; if a deployment uses a different schema, a variable substitution step would be needed. |
| No other env‑variables are referenced inside the file. |

If the deployment framework injects variables (e.g., via `sed` or a templating engine), ensure the placeholder for the database name is correctly replaced.

---

## 8. Suggested TODO / Improvements

1. **Encapsulate the TCL sector ID logic**  
   - Create a reusable Hive UDF or a small lookup table (`tcl_sector_mapping`) and replace the inline `decode(..., CAST(split_part(...)))` expression. This improves maintainability and makes unit testing easier.

2. **Add explicit partition pruning**  
   - If `mnp_portinout_details_raw` is partitioned by `portstartdate` (or month), modify the CTE to reference the partition column directly, e.g.:  
     ```sql
     FROM mnaas.mnp_portinout_details_raw PARTITION (month = ${month})
     ```
   - This will dramatically reduce scan size for large datasets.

---