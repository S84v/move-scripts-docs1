**File:** `mediation-ddls\mnaas_ra_calldate_med_data.hql`  

---

## 1. High‑Level Summary
This script creates the Hive view `mnaas.ra_calldate_med_data`. The view aggregates raw CDR (Call Detail Record) traffic data by month, usage type (Data, Voice, SMS, A2P), and customer (derived from `org_details`). It calculates the number of CDRs and total usage (seconds for voice/data, count‑1 for SMS) per customer and usage type, exposing a compact, reporting‑ready dataset for downstream billing, analytics, or reconciliation processes.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `filtered_cdr` | CTE (Common Table Expression) | Normalises raw traffic rows: extracts `month` (YYYY‑MM) from `partition_date`, derives `prefix` from the first three characters of `filename`, computes `usage` (seconds for Data/Voice, constant 1 for SMS), and classifies `usage_type` (Data, Voice, SMS, or A2P for specific MO SMS). |
| `mnaas.ra_calldate_traffic_table` | Source table | Holds the raw, partitioned CDR records that feed the view. |
| `mnaas.org_details` | Lookup table | Provides customer names (`orgname` or `companyname`) keyed by `orgno`. |
| `ra_calldate_med_data` | Hive **VIEW** | Exposes aggregated columns: `month`, `usage_type`, `tcl_secs_id`, `cust_name`, `cdr_count`, `cdr_usage`. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.ra_calldate_traffic_table` (columns: `partition_date`, `filename`, `calltype`, `retailduration`, `traffictype`, `callingparty`, `tcl_secs_id`, …) <br> - `mnaas.org_details` (columns: `orgno`, `orgname`, `companyname`) |
| **Outputs** | - Hive **VIEW** `mnaas.ra_calldate_med_data` (no physical data written; queryable like a table). |
| **Side Effects** | - DDL operation: creates or replaces the view in the `mnaas` database. No data mutation. |
| **Assumptions** | - Hive/Impala engine with `CREATE VIEW` support. <br> - `partition_date` format `YYYY-MM-DD…` (first 7 chars represent month). <br> - `filename` always at least 3 characters; prefixes `HOL` and `SNG` map to organization names. <br> - `calltype` values limited to `'Data'`, `'Voice'`, `'SMS'`. <br> - For A2P detection: `calltype='SMS'`, `traffictype='MO'`, `callingparty='31637089900'`. |
| **External Services** | - Hive Metastore (for view registration). <br> - Underlying HDFS storage where source tables reside. |

---

## 4. Integration Points  

| Connected Component | How It Links |
|---------------------|--------------|
| **`ra_calldate_traffic_table` creation scripts** (e.g., `mnaas_ra_calldate_traffic_table.hql`) | Provides the raw CDR data that this view consumes. |
| **`org_details` DDL** (`mnaas_org_details.hql`) | Supplies customer name mapping; any schema change here directly impacts the view. |
| **Down‑stream reporting scripts** (e.g., monthly usage roll‑ups, billing extracts) | Likely reference `ra_calldate_med_data` to generate customer‑level usage summaries. |
| **Reject/clean‑up views** (`*_reject.hql` series) | May feed filtered rows into `ra_calldate_traffic_table`; thus indirectly affect the view’s row set. |
| **Orchestration layer** (e.g., Airflow, Oozie, custom scheduler) | Executes this DDL as part of a “build‑views” job after the traffic table is refreshed. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Source schema drift** (e.g., column rename or type change in `ra_calldate_traffic_table`) | View creation fails or returns wrong results. | Add a pre‑deployment schema validation step (e.g., `DESCRIBE` checks) and version‑lock the view to a known schema. |
| **Missing `org_details` rows** (no matching `orgno`) | `cust_name` becomes `NULL`, aggregation may be misleading. | Ensure a nightly data‑quality job that flags orphan `tcl_secs_id` values; optionally use `COALESCE` to default to a placeholder. |
| **Performance degradation** (full table scan on large traffic table) | Long view creation time, downstream query slowness. | Partition `ra_calldate_traffic_table` by `partition_date` (already present) and add `WHERE partition_date >= …` filters in downstream queries; consider materialising the view as a table with daily refreshes. |
| **Hard‑coded A2P detection** (`callingparty='31637089900'`) | Future changes to the A2P sender number break classification. | Externalise the list of A2P numbers to a reference table (`a2p_numbers`) and join instead of hard‑coding. |
| **Decode/CASE misuse on NULL values** | Unexpected `NULL` in `usage` or `usage_type`. | Wrap columns with `NVL`/`COALESCE` where appropriate, and add unit tests for edge cases. |

---

## 6. Execution & Debugging Guide  

1. **Run the script** (typical in a CI/CD or scheduler job):  
   ```bash
   hive -f mediation-ddls/mnaas_ra_calldate_med_data.hql
   # or using beeline:
   beeline -u jdbc:hive2://<host>:10000/default -f mediation-ddls/mnaas_ra_calldate_med_data.hql
   ```

2. **Verify creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.ra_calldate_med_data;
   ```

3. **Test query** (sample):  
   ```sql
   SELECT month, usage_type, cust_name, cdr_count, cdr_usage
   FROM mnaas.ra_calldate_med_data
   WHERE month = '2024-09'
   LIMIT 20;
   ```

4. **Debugging tips**:  
   - If the view fails to create, run the inner SELECTs step‑by‑step: first `SELECT * FROM mnaas.ra_calldate_traffic_table LIMIT 10;` then the CTE query, then the final aggregation.  
   - Check for `NULL` in `partition_date` or `filename` that would break `substr`/`substring`.  
   - Use `EXPLAIN` on the final SELECT to see scan patterns and verify partition pruning.  

5. **Log handling**: The Hive CLI writes to stdout/stderr; capture logs in the orchestration system (e.g., Airflow task logs) for audit.

---

## 7. External Configuration & Environment Variables  

The script itself does **not** reference external config files or environment variables. It assumes:

- The Hive database `mnaas` exists and is the current context (or fully qualified names are used).  
- Underlying tables (`ra_calldate_traffic_table`, `org_details`) are already created and populated by other DDL scripts in the same `mediation-ddls` package.  

If the deployment environment uses a variable for the database name (e.g., `${DB_NAME}`), the calling orchestration layer should substitute it before execution.

---

## 8. Suggested Improvements (TODO)

1. **Externalise A2P sender list** – replace the hard‑coded `callingparty='31637089900'` with a join to a reference table (`mnaas.a2p_senders`). This makes the logic data‑driven and easier to maintain.  

2. **Add explicit column list to the view** – instead of `SELECT *` inside the CTE, enumerate required columns. This protects the view from accidental schema changes in the source table and improves readability.  

---