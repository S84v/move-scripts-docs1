**File:** `mediation-ddls\mnaas_ra_calldate_proposition_reject.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.ra_calldate_proposition_reject`. The view isolates call‑detail‑record (CDR) rows that lack a *proposition addon* value, are not inter‑connect traffic, and meet several file‑naming constraints. It aggregates CDR count and usage per file, month, organization, and country, and tags each row with the fixed reason “Proposition is missing”. The view is part of a family of “reject” views used for data‑quality validation and downstream reporting in the mediation layer.

---

## 2. Core Objects & Responsibilities

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_calldate_proposition_reject` | Hive **VIEW** | Presents a summarized, filtered set of traffic records that are missing a proposition addon, together with a diagnostic reason. |
| `mnaas.ra_calldate_traffic_table` | Hive **TABLE** | Source CDR data (raw or pre‑processed) containing columns referenced in the view (e.g., `filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname`, `proposition_addon`, `usagefor`, `file_prefix`, `row_num`, `calltype`, `bytes_sec_sms`, `cdr_count`). |
| `mnaas.org_details` | Hive **TABLE** | Provides organization metadata (`orgno` → `tcl_secs_id`, `orgname`). Used for a left outer join to enrich traffic rows with organization names. |
| `createtab_stmt` | Variable (script placeholder) | Holds the DDL statement; the script likely runs this variable via a wrapper (e.g., `hive -f`). |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.ra_calldate_traffic_table` (must contain the columns used in SELECT).<br>- `mnaas.org_details` (must contain `orgno` and `orgname`). |
| **Outputs** | - Hive **VIEW** `mnaas.ra_calldate_proposition_reject` (persisted in the `mnaas` database). |
| **Side‑Effects** | - Overwrites the view if it already exists (standard Hive `CREATE VIEW AS SELECT`). No data mutation occurs. |
| **Assumptions** | - Hive/Impala engine is available and the `mnaas` database is accessible.<br>- The column `row_num` exists and is pre‑populated (likely from a window function in an upstream step).<br>- `proposition_addon` is a string column that is empty (`''`) when missing.<br>- `usagefor`, `file_prefix`, and `filename` follow the naming conventions used across the mediation suite. |
| **External Services** | - None directly; the script runs inside the data‑warehouse environment. Connection details (JDBC URL, Kerberos tickets, etc.) are supplied by the job scheduler (e.g., Oozie, Airflow) or a wrapper script. |

---

## 4. Integration Points & Call Flow

| Point | Description |
|-------|-------------|
| **Upstream** | - A prior ETL step populates `ra_calldate_traffic_table` (likely from raw CDR ingestion, de‑duplication, and enrichment).<br>- The `row_num = 1` filter suggests a windowing step that assigns a row number per grouping; that step must run before this view is created. |
| **Sibling Reject Views** | - Mirrors the pattern of other reject views (`*_billing_reject`, `*_country_reject`, `*_duplicate_reject`, etc.) that each filter on a different data‑quality rule. These views are typically UNION‑ed or processed together by a reporting job that generates “reject” dashboards or alerts. |
| **Downstream** | - Reporting jobs (e.g., daily QA dashboards, SLA monitoring) query this view to count missing‑proposition records.<br>- Alerting scripts may compare the row count against thresholds and raise tickets if the volume spikes.<br>- A final “rejection export” job may write the view’s contents to an SFTP location or a Kafka topic for consumption by the billing system. |
| **Execution Wrapper** | - The file is likely invoked by a shell or Python wrapper that sets `createtab_stmt` and runs `hive -e "$createtab_stmt"` or similar. The wrapper may also log execution status to a monitoring table. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema Drift** – Underlying tables change (column rename, type change). | View creation fails; downstream jobs break. | Add a pre‑deployment schema validation step (e.g., `DESCRIBE` checks) and version‑lock the view to a specific table schema. |
| **Performance Degradation** – Large join without proper partitioning or statistics. | Long view creation time, possible OOM on Hive server. | Ensure `ra_calldate_traffic_table` is partitioned by `callmonth` and `orgno`; collect statistics (`ANALYZE TABLE … COMPUTE STATISTICS`). |
| **Incorrect Filtering** – `proposition_addon = ''` may miss NULL values. | Some missing‑proposition rows slip through, leading to inaccurate QA metrics. | Update filter to `proposition_addon IS NULL OR proposition_addon = ''`. |
| **Row‑Number Dependency** – If `row_num` is not populated, the view returns no rows. | Silent data loss. | Verify upstream step that generates `row_num` runs successfully; add a sanity check (`SELECT COUNT(*) FROM … WHERE row_num IS NULL`). |
| **View Overwrite Without Audit** – Re‑creating the view erases metadata about previous definitions. | Loss of change history. | Store the DDL in a version‑controlled repository and log each execution (timestamp, user, git commit). |

---

## 6. Running / Debugging the Script

1. **Standard Execution** (via wrapper or manually)  
   ```bash
   hive -e "
   CREATE VIEW mnaas.ra_calldate_proposition_reject AS
   SELECT filename,
          processed_date,
          callmonth,
          tcl_secs_id,
          orgname,
          '' AS proposition_addon,
          country,
          usage_type,
          SUM(cdr_count) AS cdr_count,
          SUM(
              CASE
                  WHEN calltype = 'Data'  THEN bytes_sec_sms / (1024 * 1024)
                  WHEN calltype = 'Voice' THEN bytes_sec_sms / 60
                  ELSE bytes_sec_sms
              END) AS cdr_usage,
          'Proposition is missing' AS reason
   FROM mnaas.ra_calldate_traffic_table
   LEFT OUTER JOIN mnaas.org_details ON tcl_secs_id = orgno
   WHERE tcl_secs_id > 0
     AND proposition_addon = ''
     AND usagefor != 'Interconnect'
     AND file_prefix != 'SNG'
     AND NOT filename LIKE '%NonMOVE%'
     AND row_num = 1
   GROUP BY filename, processed_date, callmonth, usage_type,
            tcl_secs_id, orgname, country;
   "
   ```

2. **Debugging Steps**  
   - **Validate source data**:  
     ```sql
     SELECT COUNT(*) FROM mnaas.ra_calldate_traffic_table WHERE proposition_addon = '' LIMIT 10;
     ```  
   - **Run the SELECT alone** (without `CREATE VIEW`) to inspect row count and sample rows.  
   - **Explain plan**: `EXPLAIN SELECT …` to verify join and filter push‑down.  
   - **Check view existence**: `SHOW VIEWS IN mnaas LIKE 'ra_calldate_proposition_reject';`  
   - **Inspect view definition**: `SHOW CREATE VIEW mnaas.ra_calldate_proposition_reject;`

3. **Operator Checklist**  
   - Confirm that the upstream traffic table load completed successfully (no errors in the previous job).  
   - Verify that the Hive metastore is reachable.  
   - After execution, run a quick count: `SELECT COUNT(*) FROM mnaas.ra_calldate_proposition_reject;` and compare against expected thresholds.

---

## 7. External Config / Environment Variables

| Variable / File | Usage |
|-----------------|-------|
| `HIVE_CONF_DIR` / `hive-site.xml` | Provides JDBC URL, authentication (Kerberos keytab), and metastore connection details for the Hive client. |
| Scheduler environment (e.g., Airflow `env` dict) | May inject variables such as `RUN_DATE` used by upstream jobs; this script itself does not reference them directly. |
| Version‑control repository (Git) | Stores the `.hql` file; the wrapper may log the current commit hash for auditability. |

*If the wrapper script defines `createtab_stmt` dynamically (e.g., templating with Jinja), note that the placeholder is replaced before execution.*

---

## 8. Suggested Improvements (TODO)

1. **Handle NULL proposition values** – Change the filter to `proposition_addon IS NULL OR proposition_addon = ''` to capture all missing cases.  
2. **Add a comment block** at the top of the file describing purpose, author, and version, and include a `DROP VIEW IF EXISTS` guard to make the script idempotent in environments where view recreation is required.  

---