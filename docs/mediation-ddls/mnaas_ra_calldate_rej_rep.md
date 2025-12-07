**File:** `mediation-ddls\mnaas_ra_calldate_rej_rep.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.ra_calldate_rej_rep`. The view aggregates all “reject” records produced by the mediation layer by UNION‑ing the ten source reject tables/views (`ra_calldate_*_reject`). The resulting view provides a single, flat representation of rejected CDRs with common columns (`filename, processed_date, callmonth, tcl_secs_id, orgname, proposition_addon, country, cdr_count, cdr_usage, reason, usage_type`). Down‑stream reporting, billing validation, and audit jobs query this view instead of each individual reject source.

---

## 2. Key Objects Defined in the File
| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.ra_calldate_rej_rep` | **VIEW** | Consolidates reject records from all mediation‑level reject tables into a unified result set for reporting and downstream processing. |
| `createtab_stmt` (internal label) | **SQL statement container** | Holds the `CREATE VIEW … AS SELECT … UNION …` DDL that is executed by the orchestration engine. |

*No procedural code, functions, or classes are defined – the file is a pure DDL definition.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs (source objects)** | Ten source reject tables/views, all residing in the `mnaas` schema:<br>• `ra_calldate_geneva_reject`<br>• `ra_calldate_ic_geneva_reject`<br>• `ra_calldate_secsid_reject`<br>• `ra_calldate_billing_reject`<br>• `ra_calldate_proposition_reject`<br>• `ra_calldate_ic_partner_reject`<br>• `ra_calldate_duplicate_reject`<br>• `ra_calldate_telesur_reject`<br>• `ra_calldate_country_reject`<br>• `ra_calldate_tadig_reject`<br>• `ra_calldate_imsi_reject`<br>• `ra_calldate_sms_reject` |
| **Outputs** | The view `mnaas.ra_calldate_rej_rep`. No physical tables are written; the view is a logical construct. |
| **Side Effects** | - Registers the view in the Hive metastore (or Impala catalog).<br>- Overwrites an existing view with the same name (implicit `CREATE OR REPLACE` semantics depending on the execution engine). |
| **Assumptions** | - All source reject objects exist and share **identical column order and data types** as listed in the SELECT clause.<br>- The execution environment has sufficient permissions to create/replace views in the `mnaas` database.<br>- No partitioning or clustering is required for the view (it is a simple UNION).<br>- The underlying tables are refreshed **before** this view is (re)created, otherwise the view may expose stale data. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Direction | Connected Component | Interaction |
|-----------|---------------------|-------------|
| **Upstream** | Individual reject‑generation scripts (e.g., `mnaas_ra_calldate_geneva_reject.hql`, `mnaas_ra_calldate_ic_partner_reject.hql`, etc.) | Those scripts populate the source tables/views that this view unions. They are typically scheduled earlier in the nightly mediation pipeline. |
| **Downstream** | Reporting jobs, audit pipelines, and billing exception handlers | Consumers run queries such as `SELECT * FROM mnaas.ra_calldate_rej_rep WHERE reason='XYZ'` to generate exception reports, SLA dashboards, or to feed corrective actions back to the OSS/BSS. |
| **Orchestration** | Workflow engine (e.g., Oozie, Airflow, Azkaban) | The DDL is invoked as a task after all reject‑generation tasks have succeeded. The task may be named `create_ra_calldate_rej_rep_view`. |
| **Metadata** | Hive Metastore / Impala Catalog | The view definition is stored here; any schema‑evolution tools (e.g., schema‑registry scripts) must be aware of the view to avoid accidental drops. |
| **Operational Monitoring** | Alerting system (e.g., PagerDuty, Splunk) | A health check may run `SHOW CREATE VIEW mnaas.ra_calldate_rej_rep` and verify that the view exists and contains the expected number of UNION branches. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source schema drift** – a new column added to one reject table but not to the others. | View creation fails or produces mismatched column order, leading to downstream query errors. | Enforce a schema‑validation step before view creation (e.g., compare `DESCRIBE` of each source). Add unit tests in CI that verify column parity. |
| **Missing source table** – a reject table not yet created (e.g., new partner reject not deployed). | `CREATE VIEW` fails, pipeline stops, downstream reporting unavailable. | Guard the DDL with `IF EXISTS` checks or generate the view dynamically based on a catalog of available tables. |
| **Performance degradation** – UNION of many large tables may cause long view materialization time or slow query performance. | Increased job runtime, possible timeouts. | Consider converting the view to a **materialized view** or a **partitioned table** refreshed nightly. Add appropriate indexes/partitioning on common filter columns (`callmonth`, `reason`). |
| **Accidental view overwrite** – a developer runs the script in a non‑prod environment and overwrites a production view. | Data inconsistency, loss of production‑specific view definition. | Use environment‑specific schema prefixes (e.g., `mnaas_dev`, `mnaas_prod`) or enforce a naming convention. Require explicit `CREATE OR REPLACE VIEW` only in controlled CI/CD pipelines. |
| **Permission issues** – the execution user lacks `CREATE VIEW` rights. | Job fails silently, alerts not triggered. | Include a pre‑flight permission check (`SHOW GRANT`) and fail fast with a clear error message. |

---

## 6. Example Execution & Debugging Workflow  

1. **Run the script** (typically via the orchestration engine):  
   ```bash
   hive -f mediation-ddls/mnaas_ra_calldate_rej_rep.hql
   # or
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_ra_calldate_rej_rep.hql
   ```

2. **Verify creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.ra_calldate_rej_rep;
   ```

3. **Quick sanity check** (sample rows):  
   ```sql
   SELECT * FROM mnaas.ra_calldate_rej_rep LIMIT 10;
   ```

4. **Validate column consistency** (run once after any upstream change):  
   ```sql
   DESCRIBE mnaas.ra_calldate_geneva_reject;
   DESCRIBE mnaas.ra_calldate_ic_partner_reject;
   -- compare output manually or via a script
   ```

5. **Debug failures**:  
   - Check Hive/Impala logs for “Table not found” or “Column mismatch”.  
   - Ensure all upstream reject tables are present (`SHOW TABLES LIKE 'ra_calldate_*_reject'`).  
   - If the view exists but returns empty results, verify that upstream jobs have populated the source tables for the current `callmonth`.  

---

## 7. External Configuration / Environment Variables  

| Config Item | Usage |
|-------------|-------|
| `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Determines the client configuration (metastore URI, authentication). |
| `ENV` (e.g., `DEV`, `PROD`) | Often used by the orchestration wrapper to select the target schema (`mnaas_dev` vs `mnaas`). The script itself hard‑codes `mnaas`, so the wrapper must substitute the schema name if needed. |
| `HADOOP_USER_NAME` | The Hadoop user under which the DDL is executed; must have `CREATE VIEW` privileges. |
| No explicit in‑file variables are referenced; the script relies on the execution context to provide connectivity and permissions. |

---

## 8. Suggested TODO / Improvements  

1. **Add schema‑validation step** – before creating the view, programmatically verify that all source reject tables have the exact column list (order & type). This can be a small pre‑script written in Bash/Python that aborts the job if a mismatch is detected.  

2. **Convert to a materialized view or nightly snapshot table** – if query performance becomes a bottleneck, replace the logical view with a pre‑aggregated table refreshed after all reject sources are loaded. Include partitioning on `callmonth` and `reason` to accelerate downstream reporting.  

--- 

*End of documentation.*