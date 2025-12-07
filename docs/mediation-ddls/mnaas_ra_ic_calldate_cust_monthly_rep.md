**File:** `mediation-ddls\mnaas_ra_ic_calldate_cust_monthly_rep.hql`  

---

## 1. High‑Level Summary
This script creates the Hive/Impala view **`mnaas.ra_ic_calldate_cust_monthly_rep`**.  
The view aggregates inter‑connect traffic (SMS and voice) for each customer (identified by `tcl_secs_id`) on a monthly basis, joining three data sources:

* **Mediated traffic** (`ra_calldate_traffic_table`) – only rows where `usagefor = 'Interconnect'`.  
* **Rejected traffic** (`ra_calldate_rej_tbl` + `month_reports`) – rejected CDRs for the same month.  
* **Generated (successful) traffic** (`ra_calldate_gen_succ_rej` + `month_reports`) – successful generated CDRs for the same month.  

The view also enriches the result with the customer’s organization name from `org_details`. It is used downstream by reporting jobs that produce monthly inter‑connect usage and revenue reports.

---

## 2. Important Objects & Their Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_ic_calldate_cust_monthly_rep` | **VIEW** | Exposes a consolidated, month‑level snapshot of mediated, rejected, and generated SMS/voice usage per customer. |
| `med_data` (CTE) | **SELECT** | Filters mediated traffic for inter‑connect usage and splits SMS vs. voice counts/bytes. |
| `med_summary` (CTE) | **AGGREGATE** | Summarises `med_data` by `callmonth` and `tcl_secs_id`. |
| `rej_data` (CTE) | **SELECT** | Pulls rejected CDRs for the target month (`bill_month = month`). |
| `rej_summary` (CTE) | **AGGREGATE** | Summarises rejected counts/usage per customer/month. |
| `gen_data` (CTE) | **SELECT** | Pulls successfully generated CDRs for the target month (`status='SUCCESS'`). |
| `gen_summary` (CTE) | **AGGREGATE** | Summarises generated counts/usage per customer/month. |
| `org_details` | **TABLE** | Provides the human‑readable organization name (`orgname`) for each `tcl_secs_id`. |
| `month_reports` | **TABLE** | Supplies the current processing month (`bill_month`) and the macro/variable `month` used in the query. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs (source tables/views)** | `mnaas.ra_calldate_traffic_table`, `mnaas.ra_calldate_rej_tbl`, `mnaas.ra_calldate_gen_succ_rej`, `mnaas.month_reports`, `mnaas.org_details` |
| **Key Columns Used** | `callmonth`, `tcl_secs_id`, `secs_id`, `usage_type`, `cdr_count`, `bytes_sec_sms`, `cdr_usage`, `status`, `bill_month`, `orgno`, `orgname` |
| **Output** | Hive/Impala **VIEW** `mnaas.ra_ic_calldate_cust_monthly_rep` (no physical table created). |
| **Side‑Effects** | None – DDL only creates/overwrites the view definition. |
| **Assumptions** | * The variable/column `month` is available in the session (e.g., set via `SET hivevar:month='202312';`). <br>* All source tables exist and contain the expected schema. <br>* `bytes_sec_sms` for voice usage is stored in seconds; the view converts voice usage to minutes (`/ 60`). <br>* `tcl_secs_id` and `secs_id` are comparable (both integer after cast). |
| **External Services** | Hive/Impala metastore, underlying HDFS storage, possibly an Oozie/Airflow scheduler that injects the `month` variable. |

---

## 4. Integration Points (How This View Connects to the Rest of the System)

| Connected Component | Relationship |
|---------------------|--------------|
| **Previous DDL scripts** (`mnaas_ra_calldate_traffic_table.hql`, `mnaas_ra_calldate_rej_tbl.hql`, `mnaas_ra_calldate_gen_succ_rej.hql`, etc.) | Provide the source tables/CTEs referenced here. Those scripts are typically run earlier in the nightly pipeline to populate the raw/processed data. |
| **Monthly Reporting Jobs** (`mnaas_ra_ic_*_report.hql` or downstream Spark/MapReduce jobs) | Query this view to generate CSV/Parquet reports for billing, SLA monitoring, and partner reconciliation. |
| **Dashboard / BI Layer** (e.g., Tableau, PowerBI) | Connects to the view via a Hive/Impala ODBC/JDBC source for ad‑hoc analysis. |
| **Data Quality / Auditing Scripts** | May compare counts from `med_summary`, `rej_summary`, and `gen_summary` against expected totals. |
| **Scheduler (Oozie / Airflow)** | Executes this DDL as part of the “create‑views” stage, passing the `month` parameter from the workflow context. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or incorrect `month` variable** | View returns empty or wrong data; downstream reports break. | Enforce a pre‑execution check (`SELECT ${hivevar:month}`) and fail fast if not set. |
| **Schema drift in source tables** (e.g., column renamed, type changed) | Query fails at runtime, causing pipeline stall. | Add unit‑style schema validation steps before view creation; version‑control DDLs together with source schema definitions. |
| **Performance degradation** (large joins on `callmonth`/`tcl_secs_id`) | Long view creation time, possible OOM in Hive/Impala. | Ensure underlying tables are partitioned by `callmonth`; consider using `/*+ MAPJOIN */` hints if one side is small. |
| **Data type mismatch between `tcl_secs_id` and `secs_id`** | Incorrect joins leading to missing rows. | Explicitly cast both to the same type (already done for `secs_id`); add a validation query after view creation to compare row counts with source tables. |
| **Incorrect handling of voice usage units** (`bytes_sec_sms` used for voice, then divided by 60) | Mis‑reported voice minutes. | Verify with data owners that `bytes_sec_sms` indeed stores seconds for voice; add a comment in the view definition. |
| **View overwrites without versioning** | Historical queries may break if view definition changes. | Adopt a naming convention with version suffix (e.g., `ra_ic_calldate_cust_monthly_rep_v1`) and keep old versions for audit. |

---

## 6. Example Execution & Debugging Workflow  

1. **Set the processing month** (e.g., December 2023):  
   ```sql
   SET hivevar:month='202312';
   ```

2. **Run the DDL** (via Hive CLI, Beeline, or Airflow task):  
   ```bash
   hive -f mediation-ddls/mnaas_ra_ic_calldate_cust_monthly_rep.hql
   ```

3. **Validate the view**:  
   ```sql
   SELECT COUNT(*) AS rows,
          SUM(med_sms_cdr) AS med_sms,
          SUM(rej_sms_cdr) AS rej_sms,
          SUM(gen_sms_cdr) AS gen_sms
   FROM mnaas.ra_ic_calldate_cust_monthly_rep
   WHERE callmonth = ${hivevar:month};
   ```

4. **Debugging tips**  
   * If the view fails to compile, check the Hive log for the exact line; most errors stem from missing columns or the `month` variable.  
   * To isolate a problem, run each CTE individually (copy‑paste the CTE query into a temporary view or `SELECT` statement).  
   * Use `EXPLAIN` on the final SELECT to verify that partitions are being pruned correctly.  

5. **Running from a scheduler** (e.g., Airflow):  
   ```python
   HiveOperator(
       task_id="create_ic_monthly_view",
       hql="mediation-ddls/mnaas_ra_ic_calldate_cust_monthly_rep.hql",
       hive_cli_conn_id="hive_default",
       params={"month": "{{ ds_nodash[:6] }}"}  # ds_nodash = 20231201 -> month = 202312
   )
   ```

---

## 7. External Configuration / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `hivevar:month` (or `${month}` in the query) | Determines which billing month’s data is aggregated. Must be supplied by the calling process (scheduler, manual run, or wrapper script). |
| Hive/Impala connection settings (e.g., `hive.metastore.uris`, `hive.exec.dynamic.partition`) | Not defined in this file but required for successful execution; typically sourced from the cluster’s `hive-site.xml`. |
| Optional: `spark.sql.shuffle.partitions` (if the view is later consumed by Spark) | Controls parallelism for downstream jobs; not directly used here but relevant for performance. |

---

## 8. Suggested TODO / Improvements  

1. **Parameterize the month via a Hive macro** – replace the back‑ticked `` `month` `` reference with `${hivevar:month}` throughout the CTEs to make the dependency explicit and avoid accidental column name collisions.  
2. **Add documentation comments** inside the view definition (e.g., `/*` blocks) describing each column, especially the conversion of voice usage (`/ 60`). This aids future maintainers and reduces the risk of unit‑misinterpretation.  

---