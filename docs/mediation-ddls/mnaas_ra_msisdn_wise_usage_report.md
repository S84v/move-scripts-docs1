**File:** `mediation-ddls\mnaas_ra_msisdn_wise_usage_report.hql`  

---

## 1. High‑Level Summary
This script creates the Hive/Impala view `mnaas.ra_msisdn_wise_usage_report`. The view aggregates raw CDR data from `mnaas.traffic_details_raw_daily` into daily, per‑MSISDN usage metrics: number of data, voice, and SMS CDRs, total data volume (MB), voice minutes, and SMS count. It is a downstream reporting artifact used by downstream analytics, billing reconciliation, and operational dashboards.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_msisdn_wise_usage_report` | **View** | Provides a pre‑aggregated, day‑level usage snapshot per MSISDN for the three service types (Data, Voice, SMS). |
| `cdr` (CTE) | **Common Table Expression** | Normalises raw CDR rows into separate columns for data usage, SMS count, voice usage, and per‑type counters, simplifying the final aggregation. |
| `mnaas.traffic_details_raw_daily` | **Source Table** | Holds the raw, daily CDR records (fields: `msisdn`, `calldate`, `calltype`, `retailduration`). All downstream usage reports depend on this table. |

*No procedural code, functions, or classes are defined in this file; the view definition is the sole artifact.*

---

## 3. Data Flow, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | `mnaas.traffic_details_raw_daily` – must contain columns `msisdn` (string), `calldate` (date), `calltype` (enum: `Data`, `SMS`, `Voice`), `retailduration` (numeric, seconds). |
| **Outputs** | Hive/Impala view `mnaas.ra_msisdn_wise_usage_report`. No physical tables are created; the view is a logical query layer. |
| **Side Effects** | DDL operation: creates (or replaces) the view. If the view already exists, the statement will fail unless executed with `CREATE OR REPLACE VIEW` (not used here). |
| **Assumptions** | • The source table is partitioned by `calldate` (common in the codebase). <br>• `retailduration` is stored in seconds; conversion to MB uses a fixed 1 MB = 1024 × 1024 bytes (implies `retailduration` for Data CDRs actually represents bytes, not seconds – a known convention in this environment). <br>• No NULL values in the key columns; if they appear, the view will propagate NULLs. |
| **External Services** | Hive/Impala metastore, underlying HDFS storage for the source table. No network calls, queues, or APIs are invoked. |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **Other DDL scripts** (e.g., `mnaas_ra_calldate_traffic_table.hql`) | Create the source table `traffic_details_raw_daily`. This view must be executed **after** those tables are available. |
| **Reporting jobs** (e.g., daily usage extraction, billing reconciliation pipelines) | Query the view directly (`SELECT * FROM mnaas.ra_msisdn_wise_usage_report WHERE calldate = …`). |
| **ETL orchestration** (Oozie / Airflow DAGs) | Typically a “DDL” task that runs all view‑creation scripts in a defined order; this file is part of the “post‑load” stage. |
| **Data quality checks** | Separate validation scripts may compare aggregated counts from this view against source table row counts. |
| **Security / ACLs** | Hive/Impala role `mnaas_analyst` likely has SELECT rights on the view; DDL execution requires `CREATE` privilege on schema `mnaas`. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source table columns renamed or data types changed. | View creation fails or returns incorrect results. | Add a pre‑deployment validation step that checks column existence/types (e.g., `DESCRIBE FORMATTED mnaas.traffic_details_raw_daily`). |
| **Performance degradation** – large daily partitions cause full‑scan aggregation. | Long query times for downstream jobs. | Ensure `traffic_details_raw_daily` is partitioned by `calldate` and that the view’s query planner can prune partitions. Consider adding `WHERE calldate = …` predicates in downstream jobs. |
| **Incorrect unit conversion** – `retailduration` for Data CDRs is treated as bytes but may be seconds after a schema change. | Mis‑reported data usage (MB). | Document the unit expectation in the data model; add a sanity‑check (e.g., `MAX(data_usage) < 10^9` bytes) in a separate validation script. |
| **View recreation failure** – script re‑runs while view exists, causing “already exists” error. | Job aborts, downstream pipelines stop. | Change DDL to `CREATE OR REPLACE VIEW` or add `DROP VIEW IF EXISTS` before creation. |
| **Missing source data** – daily raw table not loaded yet. | View returns empty or partial results. | Enforce DAG dependencies: the view creation task must run after the raw data load task. |

---

## 6. Running / Debugging the Script  

1. **Typical Execution** (via Hive CLI or Impala shell)  
   ```bash
   hive -f mediation-ddls/mnaas_ra_msisdn_wise_usage_report.hql
   # or
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_ra_msisdn_wise_usage_report.hql
   ```

2. **Verification Steps**  
   - After execution, run:  
     ```sql
     DESCRIBE FORMATTED mnaas.ra_msisdn_wise_usage_report;
     SELECT COUNT(*) FROM mnaas.ra_msisdn_wise_usage_report LIMIT 10;
     ```  
   - Compare aggregated counts with a manual query on the source table for a single day to ensure correctness.

3. **Debugging Tips**  
   - If the view fails to create, check the Hive/Impala error log for “Table not found” or “Column not found”.  
   - Use `EXPLAIN` on the view query to inspect the execution plan and verify partition pruning.  
   - Temporarily replace the `CREATE VIEW` with `CREATE TABLE AS SELECT` to materialise results and inspect intermediate data.

---

## 7. External Configuration / Environment Variables  

| Config Item | Usage |
|-------------|-------|
| `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Determines the Hive/Impala client configuration (metastore URI, authentication). |
| `HADOOP_USER_NAME` | The OS user under which the DDL runs; must have `CREATE` rights on schema `mnaas`. |
| No in‑file placeholders are present; the script relies on the default Hive/Impala connection settings of the execution environment. |

---

## 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend the script with `DROP VIEW IF EXISTS mnaas.ra_msisdn_wise_usage_report;` or switch to `CREATE OR REPLACE VIEW`. This prevents failures on re‑run and simplifies CI pipelines.

2. **Add explicit NULL handling** – wrap each CASE expression with `COALESCE(..., 0)` to guarantee numeric output even if source fields are NULL, e.g.:  
   ```sql
   COALESCE(CASE WHEN calltype='Data' THEN retailduration ELSE 0 END,0) AS data_usage
   ```

   This protects downstream aggregations from unexpected NULL propagation.

---