**File:** `mediation-ddls\mnaas_ra_calldate_ic_geneva_reject.hql`  

---

## 1. High‑Level Summary
This script creates the Hive/Impala view `mnaas.ra_calldate_ic_geneva_reject`. The view surfaces *rejected* inter‑connect (IC) voice traffic records that could not be billed because the SECS (inter‑connect) record is missing in the Geneva system. It joins the rejection log (`ra_calldate_gen_succ_rej`) with the traffic detail table (`ra_calldate_traffic_table`) and enriches the result with organization details (`org_details`). The view aggregates CDR count and usage per file, month, SECS ID, and other dimensions, and tags each row with a fixed reason string.

---

## 2. Important Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_calldate_ic_geneva_reject` | **VIEW** (created) | Provides a consolidated, aggregated view of rejected IC voice usage for downstream reporting / billing exception handling. |
| `ra_calldate_gen_succ_rej` | **TABLE** (source) | Holds per‑file rejection records from the Geneva “success/reject” process. Only rows with `status='REJECTED'` are considered. |
| `ra_calldate_traffic_table` | **TABLE** (source) | Contains raw traffic metrics (CDR count, bytes/sec/SMS, usage type, etc.) for each SECS ID and month. |
| `org_details` | **TABLE** (source) | Supplies organization name (`orgname`) and other metadata keyed by `orgno` (matches `tcl_secs_id`). |
| `createtab_stmt` | **VARIABLE** (script wrapper) | Holds the DDL string that the orchestration engine executes (e.g., via Hive CLI). |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.ra_calldate_gen_succ_rej` (columns: `filename`, `processed_date`, `secs_id`, `usage_type`, `callmonth`, `bill_month`, `status`, `row_num`, `partnertype`, `country`, `tadigcode`, `imsi_reject`)<br>- `mnaas.ra_calldate_traffic_table` (columns: `tcl_secs_id`, `callmonth`, `bill_month`, `cdr_count`, `bytes_sec_sms`, `usage_type`, `proposition_addon`)<br>- `mnaas.org_details` (columns: `orgno`, `orgname`) |
| **Outputs** | - Hive/Impala **VIEW** `mnaas.ra_calldate_ic_geneva_reject` (no physical data written; queryable like a table). |
| **Side Effects** | - DDL operation: creates or replaces the view. No data mutation on source tables. |
| **Assumptions** | - All referenced tables exist and are refreshed before this view is built.<br>- Column types are compatible (e.g., `tcl_secs_id` can be cast to STRING to match `secs_id`).<br>- The `row_num = 1` filter implies a preceding window function that guarantees a single row per key in `ra_calldate_gen_succ_rej`.<br>- `upper(partnertype) = 'INGRESS'` and `country`, `tadigcode` are non‑empty for valid records.<br>- The environment runs Hive/Impala with `CREATE VIEW` permission for schema `mnaas`. |

---

## 4. Integration Points  

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Downstream reporting jobs** (e.g., daily exception dashboards) | Query the view to list rejected IC usage and trigger alerts or manual billing adjustments. |
| **Other reject‑views** (`ra_calldate_*_reject`) | This view follows the same naming convention; orchestration scripts may iterate over all `*_reject` views to generate a consolidated reject file. |
| **ETL orchestration layer** (Airflow, Oozie, custom scheduler) | The DDL string stored in `createtab_stmt` is passed to the Hive/Impala operator task. The same orchestration may also run the source table loads (`ra_calldate_gen_succ_rej`, `ra_calldate_traffic_table`). |
| **Data quality monitoring** | Metrics on row count of the view vs. source reject count are used to verify completeness. |
| **Billing exception handler** | Consumes the view to create tickets for “SECS does not exist in Geneva”. |

*Note:* The view name suggests it is part of a family of “IC Geneva reject” objects; any script that builds a final reject file will likely `UNION ALL` this view with other reject views.

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source schema drift** (e.g., column rename or type change) | View creation fails or returns wrong data. | Add schema‑validation step in the pipeline; version‑lock tables via Hive metastore. |
| **Performance degradation** (large joins, missing partitions) | Slow downstream queries, possible OOM in Hive/Impala. | Ensure `ra_calldate_traffic_table` and `ra_calldate_gen_succ_rej` are partitioned by `callmonth`/`bill_month`; add appropriate statistics (`ANALYZE TABLE`). |
| **Stale view** (source tables refreshed after view creation) | Inconsistent reject reporting. | Schedule view recreation **after** source table loads; use a “create or replace view” pattern in the same transaction if supported. |
| **Incorrect filtering** (`row_num = 1` assumption) | Duplicate or missing reject rows. | Verify upstream window logic; add unit test that checks row count against expected. |
| **Permission issues** (lack of CREATE VIEW rights) | Pipeline aborts. | Grant `CREATE VIEW` on schema `mnaas` to the ETL service account; audit regularly. |

---

## 6. Example Execution & Debugging  

### Running the Script
```bash
# Assuming Hive CLI; replace with impala-shell if needed
hive -hiveconf hive.exec.dynamic.partition=true \
     -f mediation-ddls/mnaas_ra_calldate_ic_geneva_reject.hql
```
*In most production pipelines the file is not executed directly; the orchestration engine extracts the `createtab_stmt` string and runs it via a Hive/Impala operator.*

### Verifying the View
```sql
-- Quick sanity check
SELECT COUNT(*) AS reject_rows,
       SUM(cdr_count) AS total_cdrs,
       SUM(cdr_usage) AS total_usage
FROM mnaas.ra_calldate_ic_geneva_reject;
```
*Compare the counts with the source reject table (`ra_calldate_gen_succ_rej`) filtered on the same criteria.*

### Debugging Tips
1. **Syntax errors** – Run the DDL in an interactive Hive session to get the exact error line.  
2. **Empty result** – Remove filters step‑by‑step (e.g., drop `country != ''`) to locate overly restrictive predicates.  
3. **Join mismatches** – Verify that `CAST(t.tcl_secs_id AS STRING)` matches the format of `r.secs_id`. Use `SELECT DISTINCT secs_id FROM ra_calldate_gen_succ_rej LIMIT 10;` to inspect.  
4. **Performance** – Run `EXPLAIN` on the view query to see join order and whether partitions are being pruned.

---

## 7. External Configuration / Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Determines Hive/Impala client configuration (metastore URI, authentication). |
| `DB_SCHEMA=mnaas` (often set via a variable) | Allows the same script to be reused for different schemas in dev/test environments. |
| `CREATE_VIEW_REPLACE` flag (custom) | Some orchestration wrappers replace the view if it exists; otherwise they may `DROP VIEW` first. |
| **No hard‑coded file paths** – the script only references tables within the `mnaas` database. |

If the production environment injects additional session variables (e.g., `hive.exec.dynamic.partition.mode=nonstrict`), they must be set before execution.

---

## 8. Suggested Improvements (TODO)

1. **Add explicit partition pruning** – Modify the view to include `WHERE t.callmonth = ${processing_month}` (parameterized) so that downstream queries automatically limit to the current month, reducing scan size.
2. **Document column lineage** – Create a small markdown file (or Hive metastore comment) that maps each output column (`cdr_count`, `cdr_usage`, `reason`) back to its source column(s). This aids impact analysis when upstream tables change.  

---