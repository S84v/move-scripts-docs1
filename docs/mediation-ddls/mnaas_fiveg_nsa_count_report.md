**File:** `mediation-ddls\mnaas_fiveg_nsa_count_report.hql`  

---

## 1. High‑Level Summary
This script defines a Hive/Impala view `mnaas.fiveg_nsa_count_report`. The view aggregates distinct 5G Non‑Standalone (NSA) product subscriptions per month, per TCL (Telecom Carrier) sector, and per organization name. It counts active subscribers by evaluating activation/de‑activation dates against the reporting period defined in `month_reports`. The view is intended for downstream reporting, billing, and KPI calculations that need a per‑org subscriber count for 5G NSA services.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `fiveg_nsa_count_report` | **VIEW** (in schema `mnaas`) | Provides a pre‑aggregated, month‑level count of distinct ICCIDs (subscriber identifiers) for active 5G NSA subscriptions, grouped by month, TCL sector ID, and organization name. |
| `mnaas.nsa_fiveg_product_subscription` | **TABLE** (source) | Holds subscription records for 5G NSA products, including `iccid`, `activation_date`, `deactivation_date`, and `tcl_secs_id`. |
| `mnaas.month_reports` | **TABLE** (source) | Supplies the reporting window (`month`, `start_with_time`, `end_with_time`). |
| `mnaas.org_details` | **TABLE** (source) | Maps `orgno` to human‑readable `orgname`. |

*No procedural code, functions, or classes are defined in this file; the only artifact is the view definition.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.nsa_fiveg_product_subscription` (columns: `iccid`, `activation_date`, `deactivation_date`, `tcl_secs_id`)<br>- `mnaas.month_reports` (columns: `month`, `start_with_time`, `end_with_time`)<br>- `mnaas.org_details` (columns: `orgno`, `orgname`) |
| **Outputs** | - Hive/Impala view `mnaas.fiveg_nsa_count_report` exposing columns: `month`, `tcl_secs_id`, `orgname`, `subs_count` |
| **Side Effects** | - DDL operation: `CREATE VIEW`. If the view already exists, the script will fail unless run with `CREATE OR REPLACE VIEW` (not present). |
| **Assumptions** | - All source tables exist and are refreshed before this view is materialized.<br>- Date columns (`activation_date`, `deactivation_date`, `start_with_time`, `end_with_time`) are stored as comparable timestamps or strings in a format that Hive can compare directly.<br>- `deactivation_date` may be `NULL` or an empty string to indicate “still active”.<br>- `tcl_secs_id` in the subscription table matches `orgno` in `org_details` (1‑to‑1 mapping).<br>- No partitioning or clustering is required for the view itself; performance relies on underlying table statistics. |

---

## 4. Integration Points (How This View Connects to the Rest of the System)

| Connected Component | Relationship |
|---------------------|--------------|
| **Billing / Revenue Assurance jobs** (e.g., `mnaas_billing_*` scripts) | Likely consume `fiveg_nsa_count_report` to compute usage‑based charges, generate invoices, or reconcile subscriber counts. |
| **KPI / Dashboard pipelines** (e.g., reporting dashboards, BI tools) | Query the view for month‑over‑month subscriber growth, churn, or sector‑level performance. |
| **Data Quality / Validation scripts** | May compare the view’s `subs_count` against other aggregation sources (e.g., raw CDR counts) to detect anomalies. |
| **ETL orchestration (Airflow, Oozie, etc.)** | The view creation is typically part of a “DDL” stage that runs after the upstream tables are populated (e.g., after `nsa_fiveg_product_subscription` load). |
| **Archival / Snapshot processes** | Some nightly jobs may `INSERT OVERWRITE` a partitioned table from this view for historical retention. |

*Because the view is read‑only, downstream scripts reference it via standard `SELECT` statements; no explicit coupling is coded in this file.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **View creation failure due to existing view** | Job aborts, downstream pipelines stall. | Use `CREATE OR REPLACE VIEW` or add a pre‑check (`DROP VIEW IF EXISTS`). |
| **Incorrect date comparison (string vs timestamp)** | Mis‑count of active subscribers (over/under‑reporting). | Verify column data types; cast to `timestamp` if needed (`CAST(activation_date AS TIMESTAMP)`). |
| **Performance degradation on large tables** | Long query times for downstream jobs. | Ensure source tables have appropriate partitioning (e.g., by month) and statistics refreshed (`ANALYZE TABLE`). |
| **Empty string handling for `deactivation_date`** | May treat empty strings as active when they should be ignored. | Standardize `deactivation_date` to `NULL` during upstream ETL, or add explicit `deactivation_date <> ''` filter. |
| **Schema drift (renamed columns or missing tables)** | View becomes invalid, causing runtime errors. | Include schema validation step in the orchestration before view creation. |
| **Security / Privilege issues** | Unauthorized users could drop/alter the view. | Grant only `SELECT` on the view to consumer roles; restrict DDL to admin role. |

---

## 6. Execution & Debugging Guide

1. **Run the script**  
   ```bash
   hive -f mediation-ddls/mnaas_fiveg_nsa_count_report.hql
   # or impala-shell -i <impala-host> -f ...
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE VIEW mnaas.fiveg_nsa_count_report;
   ```

3. **Sample query to validate logic**  
   ```sql
   SELECT month, tcl_secs_id, orgname, subs_count
   FROM mnaas.fiveg_nsa_count_report
   WHERE month = '2024-09';
   ```

4. **Check row counts against source tables** (quick sanity check)  
   ```sql
   SELECT COUNT(DISTINCT iccid) AS raw_subs
   FROM mnaas.nsa_fiveg_product_subscription
   WHERE activation_date <= (SELECT end_with_time FROM mnaas.month_reports WHERE month='2024-09')
     AND (deactivation_date IS NULL OR deactivation_date = '' OR deactivation_date BETWEEN (SELECT start_with_time FROM mnaas.month_reports WHERE month='2024-09') AND (SELECT end_with_time FROM mnaas.month_reports WHERE month='2024-09'));
   ```

5. **Debugging tips**  
   - If the view returns fewer rows than expected, inspect `deactivation_date` values for unexpected non‑NULL strings.  
   - Use `EXPLAIN` on the view query to identify missing indexes or costly joins.  
   - Review Hive/Impala logs for parsing errors (e.g., missing backticks).  

---

## 7. External Configuration / Environment Variables

The script itself does not reference external config files or environment variables. However, its successful execution depends on:

| Config Item | Usage |
|-------------|-------|
| **Hive/Impala connection settings** (e.g., `hive.metastore.uris`, `impala.host`) | Provided by the orchestration framework (Airflow, Oozie, etc.) that launches the script. |
| **Database credentials** (Kerberos tickets, LDAP, or username/password) | Managed outside the script, typically via a secure credential store. |
| **Warehouse location / HDFS paths** | Implicitly used by the underlying tables; any change requires table recreation, not view modification. |

If the environment uses a templating engine (e.g., Jinja) to inject schema names, verify that the placeholder resolves to `mnaas`.

---

## 8. Suggested TODO / Improvements

1. **Make the view idempotent** – replace the `CREATE VIEW` with `CREATE OR REPLACE VIEW` (or add a drop‑if‑exists step) to avoid failures on re‑runs.  
2. **Add explicit data‑type casts and null handling** – e.g.,  
   ```sql
   CAST(activation_date AS TIMESTAMP) <= end_with_time
   AND (deactivation_date IS NULL OR deactivation_date = '' OR CAST(deactivation_date AS TIMESTAMP) BETWEEN start_with_time AND end_with_time)
   ```  
   This protects against string‑comparison bugs and clarifies intent.  

---