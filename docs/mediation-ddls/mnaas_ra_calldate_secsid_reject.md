**File:** `mediation-ddls\mnaas_ra_calldate_secsid_reject.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.ra_calldate_secsid_reject`. The view extracts records from the core mediation table `mnaas.ra_calldate_traffic_table` that have an invalid SECS identifier (`tcl_secs_id = 0`). It filters out test/partner files (`file_prefix != 'SNG'`, filenames not containing “NonMOVE”) and keeps only the first logical row per CDR (`row_num = 1`). The view aggregates CDR count and usage (bytes → MB for data, seconds for voice) and tags each row with the rejection reason **“Invalid SECS ID”**. Down‑stream reporting jobs consume this view to produce rejection statistics and to trigger alerts for malformed inbound traffic.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.ra_calldate_secsid_reject` | **View** | Provides a summarized, reject‑only slice of the mediation traffic table for SECS‑ID‑0 records. |
| `mnaas.ra_calldate_traffic_table` | **Source Table** | Holds raw, per‑CDR mediation data (filename, processed_date, callmonth, tcl_secs_id, etc.). |
| Columns selected / derived | – | `filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname` (empty placeholder), `proposition_addon`, `country`, `usage_type`, aggregated `cdr_count`, aggregated `cdr_usage`, constant `reason = 'Invalid SECS ID'`. |
| `sum((CASE …)) cdr_usage` | **Expression** | Normalises usage: <br>• Data → MB (`bytes_sec_sms / (1024*1024)`) <br>• Voice/ICVOICE → seconds (`bytes_sec_sms / 60`) <br>• Other → raw bytes. |
| `WHERE` clause | **Filter** | Enforces: `tcl_secs_id = 0`, `file_prefix != 'SNG'`, filename not like `%NonMOVE%`, `row_num = 1`. |
| `GROUP BY` clause | **Aggregation** | Groups by the non‑aggregated columns to produce one row per unique combination. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Source table `mnaas.ra_calldate_traffic_table` (must exist and contain the columns referenced). <br>- Implicit environment: Hive/Impala execution engine, default database `mnaas`. |
| **Outputs** | - View `mnaas.ra_calldate_secsid_reject` (replaces any existing definition). |
| **Side‑Effects** | - DDL operation (CREATE VIEW) may acquire metadata locks; concurrent DML on the source table is unaffected. |
| **Assumptions** | - `tcl_secs_id` is numeric; a value of `0` unequivocally indicates an invalid SECS ID. <br>- `file_prefix` and `filename` columns are present and follow the naming conventions used across the mediation suite. <br>- `row_num` is pre‑computed (e.g., via a window function) to guarantee a single logical row per CDR. <br>- No partitioning is required for this view; downstream jobs handle any needed partition pruning. |
| **External Services** | None directly invoked. The view is consumed by downstream batch jobs, reporting dashboards, or alerting scripts that query the `mnaas` schema. |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **Other reject‑views** (`*_reject.hql` files) | All reject‑views share the same source table and similar filtering logic; they are later UNIONed or processed together to produce a master rejection report. |
| **Monthly/Customer reporting scripts** (`mnaas_ra_calldate_*_rep.hql`) | These scripts join the reject‑views to calculate overall rejection rates per month, per customer, or per proposition. |
| **ETL orchestration (e.g., Oozie / Airflow DAG)** | The view creation script is scheduled after the raw traffic table is populated (typically the “load” stage) and before any reporting jobs that depend on rejection data. |
| **Alerting / Monitoring** | A downstream job may run `SELECT COUNT(*) FROM mnaas.ra_calldate_secsid_reject` and raise an alarm if the count exceeds a threshold. |
| **Data lineage tools** | The view definition is captured by lineage scanners (e.g., Apache Atlas) to map the flow from raw CDR ingestion → reject view → reporting. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **View recreation during peak load** – metadata lock may block concurrent queries. | Temporary query failures or increased latency. | Schedule view creation during off‑peak windows; use `CREATE OR REPLACE VIEW` (if supported) to minimise lock time. |
| **Source schema drift** – missing/renamed columns break the view. | Job failure, missing rejection data. | Add a schema‑validation step (e.g., `DESCRIBE` check) before executing the DDL; version‑control the view definition alongside the source table schema. |
| **Incorrect filter logic** – legitimate records with `tcl_secs_id = 0` may be falsely rejected. | Over‑reporting of errors, possible SLA breach. | Verify with domain SMEs that `0` is never a valid SECS ID; optionally add a whitelist of known exceptions. |
| **Data volume growth** – aggregation may become expensive if the source table is not partitioned. | Longer view creation time, possible OOM. | Consider adding a partition clause (e.g., `PARTITIONED BY (callmonth)`) to the source table and/or materialising the view as a table refreshed nightly. |
| **Missing view after accidental DROP** – downstream jobs fail. | Production outage. | Include the view creation script in the regular deployment pipeline; enable automated recreation on start‑up of dependent jobs. |

---

## 6. Running / Debugging the Script  

1. **Execution**  
   ```bash
   hive -f mediation-ddls/mnaas_ra_calldate_secsid_reject.hql
   # or, for Impala:
   impala-shell -f mediation-ddls/mnaas_ra_calldate_secsid_reject.hql
   ```

2. **Verification**  
   ```sql
   SELECT COUNT(*) AS total_rejects,
          SUM(cdr_count) AS total_cdrs,
          SUM(cdr_usage) AS total_usage_mb
   FROM mnaas.ra_calldate_secsid_reject;
   ```

3. **Debugging Tips**  
   - **Check source data**: `SELECT DISTINCT tcl_secs_id FROM mnaas.ra_calldate_traffic_table WHERE tcl_secs_id = 0 LIMIT 10;`  
   - **Validate filters**: Run the `WHERE` clause as a standalone SELECT to confirm row counts before aggregation.  
   - **Explain plan**: `EXPLAIN SELECT ... FROM mnaas.ra_calldate_traffic_table WHERE ...;` to ensure the query uses appropriate partitions/indices.  
   - **Log capture**: Enable Hive/Impala query logging (`set hive.exec.verbose=true;`) to capture any parsing errors.

---

## 7. External Configuration & Environment Variables  

The script does **not** reference external configuration files or environment variables directly. It relies on:

| Item | Source |
|------|--------|
| Database/catalog name (`mnaas`) | Implicit in Hive/Impala session (set via `USE mnaas;` or default). |
| Table/column definitions | Managed by the overall mediation schema; any changes must be coordinated across the suite of DDL scripts. |

If the deployment environment uses a variable for the schema (e.g., `${SCHEMA}`), the script would need to be templated; currently it is hard‑coded.

---

## 8. Suggested Improvements (TODO)

1. **Add Idempotent Creation** – Replace the raw `CREATE VIEW` with `CREATE OR REPLACE VIEW` (or a conditional drop‑then‑create) to make the script safe to re‑run without manual cleanup.  
2. **Document Reason Code** – Store the rejection reason in a lookup table (`reject_reason_dim`) and reference it via a foreign key instead of a hard‑coded string, enabling easier addition of new reasons and consistent reporting.  

---