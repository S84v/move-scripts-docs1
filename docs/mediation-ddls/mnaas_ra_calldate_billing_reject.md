**File:** `mediation-ddls\mnaas_ra_calldate_billing_reject.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.ra_calldate_billing_reject`. The view aggregates call‑detail‑record (CDR) traffic that originates from files with the prefix **SNG** (i.e., MVNO‑originated traffic) and marks it as “Not Billable – MVNO”. For each combination of file, processing date, month, usage type, TCL/SEC ID, organization name and country, it sums the number of CDRs and converts the raw `bytes_sec_sms` metric into a usage value (MB for data, minutes for voice, raw units otherwise). The view is used downstream for reconciliation, reporting, and exclusion from billing pipelines.

---

## 2. Key Objects & Responsibilities

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_calldate_billing_reject` | Hive **VIEW** | Provides a pre‑aggregated, filtered snapshot of MVNO traffic that should be excluded from billing. |
| `mnaas.ra_calldate_traffic_table` | Hive **TABLE** (source) | Holds raw CDR traffic per file, including columns `filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname`, `country`, `usage_type`, `calltype`, `bytes_sec_sms`, `cdr_count`, `file_prefix`. |
| `mnaas.org_details` | Hive **VIEW/TABLE** (source) | Maps `tcl_secs_id` (orgno) to organization metadata (e.g., `orgname`). Defined in a previous script (`mnaas_org_details.hql`). |
| `createtab_stmt` | Variable (used by the build framework) | Holds the DDL string that the deployment engine executes. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.ra_calldate_traffic_table` (raw traffic) <br> - `mnaas.org_details` (org metadata) |
| **Outputs** | - Hive view `mnaas.ra_calldate_billing_reject` (persisted in the Metastore) |
| **Side Effects** | - DDL operation: creates/overwrites the view definition in the Hive Metastore. No data files are written directly. |
| **Assumptions** | - Columns `file_prefix`, `calltype`, `bytes_sec_sms`, `cdr_count` exist with expected data types (string, string, numeric, numeric). <br> - `calltype` values are limited to `'Data'`, `'Voice'`, or others handled by the `CASE`. <br> - The Hive/Impala engine is reachable and the `mnaas` database exists. |

---

## 4. Integration Points & Call Graph

| Connected Component | Relationship |
|---------------------|--------------|
| **Downstream scripts** (e.g., reconciliation, reporting jobs) | Query `ra_calldate_billing_reject` to exclude MVNO traffic from billing calculations or to generate “reject” reports. |
| **`mnaas_ra_calldate_traffic_table`** | Populated by upstream ingestion pipelines that parse SNG‑prefixed files from the MVNO source system (likely via SFTP/FTP drop). |
| **`mnaas_org_details`** | Provides organization names; other scripts that enrich billing data also join to this view. |
| **Job orchestration layer** (Airflow, Oozie, Control-M) | Executes this HQL as part of the “DDL refresh” stage, typically after the traffic table is refreshed for a new processing window. |
| **Metastore / HiveServer2** | Stores the view definition; any client that accesses Hive/Impala can read the view. |

*Note:* The view name follows the naming convention used by other “reject” views (e.g., `ra_calldate_traffic_reject` if present). It is therefore expected that any script that builds a billing‑ready dataset will `LEFT JOIN` or `EXCEPT` this view to filter out the rows.

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source schema change** (e.g., `bytes_sec_sms` renamed) | View creation fails; downstream jobs break. | Add a schema‑validation step before DDL execution; version‑control the view definition. |
| **Missing source tables** (`ra_calldate_traffic_table` or `org_details`) | DDL fails, job aborts. | Include a pre‑check that required tables/views exist; fail fast with clear log message. |
| **Division by zero / unexpected `calltype`** | Incorrect `cdr_usage` values (NULL or error). | Ensure `bytes_sec_sms` is non‑NULL; add a safe‑guard `CASE WHEN bytes_sec_sms IS NULL THEN 0 ELSE … END`. |
| **Stale view after source data refresh** | Downstream reports use outdated aggregates. | Schedule view recreation after each successful load of `ra_calldate_traffic_table`. |
| **Concurrent DDL execution** (multiple pipelines trying to recreate the view) | Metastore lock contention, possible view corruption. | Serialize DDL jobs via a lock file or orchestrator dependency. |

---

## 6. Running / Debugging the Script

1. **Standard execution (via orchestration)**  
   ```bash
   hive -f mediation-ddls/mnaas_ra_calldate_billing_reject.hql
   ```
   The orchestration tool (Airflow/Oozie) typically wraps this call and logs success/failure.

2. **Manual validation**  
   ```sql
   -- Verify view definition
   SHOW CREATE VIEW mnaas.ra_calldate_billing_reject;

   -- Sample data check
   SELECT * FROM mnaas.ra_calldate_billing_reject LIMIT 20;
   ```
   Look for:
   - Expected `reason` column value = “Not Billable - MVNO”.
   - Reasonable `cdr_usage` conversion (e.g., MB for Data).

3. **Debugging failures**  
   - Check HiveServer2 logs for syntax errors.  
   - Confirm that `file_prefix = 'SNG'` actually matches rows in the source table.  
   - Run the underlying SELECT alone (without `CREATE VIEW`) to isolate data issues.

---

## 7. External Configuration & Environment Variables

The script itself does **not** reference external config files or environment variables. However, the execution environment typically provides:

| Variable | Purpose |
|----------|---------|
| `HIVE_CONF_DIR` | Location of Hive configuration (metastore, JDBC URL). |
| `HADOOP_USER_NAME` | User identity for Hive/Impala access. |
| `DB_NAME` (if templated) | May be injected by the deployment framework to replace the hard‑coded `mnaas` schema. |

If the deployment framework uses a templating engine (e.g., Jinja, Velocity), verify that the `mnaas` database name is not overridden elsewhere.

---

## 8. Suggested Improvements (TODO)

1. **Idempotent DDL** – prepend the statement with `DROP VIEW IF EXISTS mnaas.ra_calldate_billing_reject;` to guarantee clean recreation during repeated runs.  
2. **Parameterize `file_prefix`** – expose the prefix (`'SNG'`) as a configurable variable so the same view definition can be reused for other prefixes without code change.

---