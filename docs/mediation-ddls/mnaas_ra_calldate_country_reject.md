**File:** `mediation-ddls\mnaas_ra_calldate_country_reject.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view **`mnaas.ra_calldate_country_reject`**. The view isolates call‑detail‑record (CDR) rows from the `ra_calldate_traffic_table` where the **country field is empty**, aggregates the CDR count and usage (converted to MB for data, minutes for voice, or raw units otherwise), and tags each row with a static rejection reason *“Country Name is missing”*. The view is used downstream for data‑quality reporting, rejection handling, and possibly for generating corrective action files.

---

## 2. Key Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_calldate_country_reject` | Hive **VIEW** | Presents a de‑duplicated, aggregated list of traffic records that lack a country code, together with a standard rejection reason. |
| `ra_calldate_traffic_table` | Hive **TABLE** (source) | Holds raw, per‑record CDR traffic data for a given processing date/month. |
| `org_details` | Hive **TABLE** (lookup) | Provides organization metadata (`orgno`, `orgname`, etc.) used for enriching the view via a left join on `tcl_secs_id`. |
| `createtab_stmt` (script variable) | Literal string | Holds the `CREATE VIEW … AS SELECT …` DDL; the script may be invoked by a higher‑level orchestration engine that extracts this variable and executes it. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.ra_calldate_traffic_table` (columns: `filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname`, `proposition_addon`, `country`, `usage_type`, `cdr_count`, `bytes_sec_sms`, `calltype`, `usagefor`, `partnertype`, `file_prefix`, `row_num`)<br>- `mnaas.org_details` (columns: `orgno`, `orgname`, …) |
| **Outputs** | - Hive view `mnaas.ra_calldate_country_reject` (columns: `filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname`, `proposition_addon`, `country`, `usage_type`, `cdr_count`, `cdr_usage`, `reason`) |
| **Side‑Effects** | - Registers/overwrites the view in the Hive metastore (no physical data written). |
| **Assumptions** | - Hive/Tez/LLAP environment is available and the `mnaas` database exists.<br>- Source tables are populated for the target processing window.<br>- `bytes_sec_sms` is numeric and non‑null for rows that pass the filter.<br>- The view is consumed only after the underlying tables are refreshed for the same processing date. |
| **External Services** | - Hive Metastore (DDL registration).<br>- Down‑stream jobs (e.g., rejection file generators, monitoring dashboards) that query this view. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| `mnaas_ra_calldate_traffic_table.hql` (or similar) | Populates the source traffic table that this view reads. |
| `mnaas_org_details.hql` | Creates/maintains the `org_details` lookup table used in the join. |
| Rejection‑handling jobs (e.g., `mnaas_reject_export.hql`) | Likely query `ra_calldate_country_reject` to generate CSV/flat‑file exports for operational teams. |
| Monitoring dashboards (e.g., Grafana/PowerBI) | May reference the view to surface “missing country” metrics. |
| Orchestration layer (Airflow, Oozie, Control-M) | Executes this script as part of a daily/weekly data‑quality validation DAG, typically after the traffic table load completes. |
| Other reject‑reason views (`ra_calldate_traffic_reject`, `ra_calldate_interconnect_reject`, …) | Parallel views that capture different validation failures; downstream aggregation may union them. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table schema change** (e.g., column rename or type change) | View creation fails; downstream jobs break. | Add schema‑validation step before view creation; version‑control DDL scripts. |
| **Large data volume** causing long aggregation time or OOM errors | Job stalls, resource contention. | Partition source tables by `callmonth`/`processed_date`; add `WHERE callmonth = '${target_month}'` filter if appropriate. |
| **Division by zero / null `bytes_sec_sms`** leading to NULL `cdr_usage` | Inaccurate usage metrics. | Coalesce `bytes_sec_sms` to 0, guard division with `CASE WHEN bytes_sec_sms IS NOT NULL`. |
| **Missing `org_details` rows** (left join returns NULL) | `orgname` may be NULL, affecting downstream reporting. | Ensure `org_details` is refreshed before this view; optionally replace NULL with a placeholder. |
| **Incorrect filtering (`country = ''`)** missing rows where country is NULL | Incomplete reject list. | Adjust filter to `country IS NULL OR country = ''`. |
| **View name collision** (existing view with different definition) | Unexpected data if view not recreated. | Use `CREATE OR REPLACE VIEW` (if supported) or drop before create. |

---

## 6. Running / Debugging the Script  

1. **Execute via Hive CLI / Beeline**  
   ```bash
   beeline -u jdbc:hive2://<hive-host>:10000/mnaas -f mediation-ddls/mnaas_ra_calldate_country_reject.hql
   ```
   The script should output “OK” and the view will appear in the metastore.

2. **Validate the view** (quick sanity check)  
   ```sql
   SELECT COUNT(*) AS total_rows,
          SUM(cdr_count) AS total_cdrs,
          SUM(cdr_usage) AS total_usage
   FROM mnaas.ra_calldate_country_reject;
   ```
   Verify that `reason` column always contains “Country Name is missing”.

3. **Debugging steps**  
   - Run the SELECT part alone (copy the inner query) to see raw rows before aggregation.  
   - Check for unexpected NULLs: `SELECT * FROM ... WHERE country IS NULL;`  
   - Review Hive execution plan: `EXPLAIN SELECT … FROM …;` to ensure joins are broadcast/partitioned as expected.

4. **Log monitoring**  
   - In the orchestration tool, capture Hive logs; look for “Error while compiling view” or “SemanticException”.  
   - Set `hive.exec.dynamic.partition.mode=nonstrict` if the view later references partitioned tables.

---

## 7. External Configuration / Environment Variables  

The script itself does not reference external config files or environment variables. However, typical deployment environments inject:

| Variable | Use |
|----------|-----|
| `HIVE_CONF_DIR` | Points to Hive configuration (metastore URI, execution engine). |
| `DB_NAME` (if templated) | Could be used by the orchestration wrapper to replace `mnaas` with a target schema. |
| `TARGET_MONTH` / `PROCESS_DATE` | May be supplied by the scheduler to limit the source data (not present in this static script). |

If the orchestration layer parameterizes the script, ensure those variables are documented in the wrapper job.

---

## 8. Suggested Improvements (TODO)

1. **Add NULL handling for the country filter**  
   ```sql
   ... AND (country IS NULL OR country = '')
   ```
   This captures rows where the column is truly missing.

2. **Convert to `CREATE OR REPLACE VIEW`** (Hive 2.1+ supports it) to make the script idempotent and avoid manual drops:
   ```sql
   CREATE OR REPLACE VIEW `mnaas`.`ra_calldate_country_reject` AS ...
   ```

   Optionally, include a comment header with author, creation date, and purpose for future maintainers.

---