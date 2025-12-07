**File:** `mediation-ddls\mnaas_billing_active_non_tolling_eids.hql`

---

### 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.billing_active_non_tolling_eids`. The view lists all *active* EIDs (equipment identifiers) for the current billing month that are **not** part of the tolling‑EID set and that belong to commercial offers (non‑empty `commercialoffer`). It also excludes a hard‑coded list of `tcl_secs_id` values (41218, 41648, 37226). The view is used downstream by billing and revenue‑recognition jobs that need to charge only non‑tolling, commercially‑offered equipment.

---

### 2. Objects Defined & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `billing_active_non_tolling_eids` | Hive **VIEW** (in schema `mnaas`) | Provides a filtered projection of `tcl_secs_id`, `commercialoffer`, and `eid` for active, billable, non‑tolling equipment. |

*No procedural code (functions, procedures) is present; the file is a pure DDL statement.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs (source tables / views)** | `mnaas.active_eid_list` – contains `eid`, `tcl_secs_id`, `month`, etc.<br>`mnaas.month_billing` – contains the current billing month identifier (`month`).<br>`mnaas.tolling_eid_list` – list of `eid`s that are subject to tolling (must be up‑to‑date). |
| **Output** | Hive **VIEW** `mnaas.billing_active_non_tolling_eids`. No physical data is written; the view materialises at query time. |
| **Side Effects** | Alters the Hive metastore: registers or replaces the view definition. If the view already exists, it is overwritten. |
| **Assumptions** | • The three source objects exist and are refreshed before this script runs (typically by daily “raw” DDL scripts such as `mnaas_active_eid_list.hql`, `mnaas_month_billing.hql`, `mnaas_tolling_eid_list.hql`).<br>• `commercialoffer` column is present and non‑null in `active_eid_list`.<br>• The hard‑coded `tcl_secs_id` exclusions are static business rules. |
| **External Services** | Hive/Impala metastore, underlying Hadoop/HDFS storage. No network calls, SFTP, or external APIs. |

---

### 4. Connectivity to Other Scripts / Components  

| Connected Script | Relationship |
|------------------|--------------|
| `mnaas_active_eid_list.hql` | Populates `mnaas.active_eid_list` used in the view’s join. |
| `mnaas_month_billing.hql` (or similar) | Supplies the `month` reference for the join condition. |
| `mnaas_tolling_eid_list.hql` | Provides the exclusion list of tolling EIDs. |
| Down‑stream billing jobs (e.g., `billing_daily_aggregation.hql`, `revenue_recognition.hql`) | Consume the view to calculate charges only for non‑tolling, commercially‑offered equipment. |
| Oozie / Airflow DAGs | Typically schedule this DDL as part of a “DDL refresh” workflow that runs after the raw tables are refreshed. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table schema drift** – if `active_eid_list` or `month_billing` change column names/types, view creation will fail. | Job failure, downstream billing impact. | Add schema validation step (e.g., `DESCRIBE` checks) before view creation; version‑control DDL scripts. |
| **Hard‑coded `tcl_secs_id` list becomes stale** – new IDs may need exclusion but are not added. | Incorrect billing (charging excluded equipment). | Store exclusion list in a reference table (`mnaas.non_billable_tcl_secs`) and join instead of inline literals. |
| **View replacement without dependency check** – downstream jobs may be running while the view is being re‑created, causing temporary “view not found” errors. | Transient job failures. | Use `CREATE OR REPLACE VIEW` (already implicit) and schedule view refresh during low‑traffic windows; optionally use a staging view and atomic rename. |
| **Stale tolling list** – if `tolling_eid_list` is not refreshed before this script, some tolling EIDs may slip into the view. | Revenue leakage. | Enforce ordering in the DAG: tolling list refresh → this view creation. |

---

### 6. Running / Debugging the Script  

1. **Typical Execution**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_active_non_tolling_eids.hql
   ```
   *In production this is invoked by an Oozie/Airflow task that runs after the raw‑table refresh tasks.*

2. **Verification Steps**  
   * After execution, run:  
     ```sql
     SHOW CREATE VIEW mnaas.billing_active_non_tolling_eids;
     SELECT COUNT(*) FROM mnaas.billing_active_non_tolling_eids LIMIT 10;
     ```  
   * Compare row counts with expectations (e.g., `SELECT COUNT(*) FROM mnaas.active_eid_list WHERE commercialoffer!=''` minus tolling count).  

3. **Debugging Tips**  
   * If the view fails to create, inspect the Hive metastore logs for “Table not found” errors – indicates a missing source table.  
   * Use `EXPLAIN` on the view query to ensure the join predicates are being applied as intended.  
   * Temporarily replace the `NOT IN (SELECT DISTINCT eid …)` clause with a `LEFT ANTI JOIN` to test performance if the tolling list is large.

---

### 7. External Configuration / Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Determines Hive/ Hadoop client configuration for the execution environment. |
| `METASTORE_URI` (if set) | Points to the Hive metastore service; required for view registration. |
| No script‑specific parameters are read; the DDL is static. |

If the deployment uses a templating system (e.g., Jinja2) to inject schema names, verify that the `mnaas` schema is correctly resolved at runtime.

---

### 8. Suggested Improvements (TODO)

1. **Externalise the exclusion list** – replace the inline `tcl_secs_id NOT IN (41218,41648,37226)` with a reference table (`mnaas.non_billable_tcl_secs`) to simplify future updates and enable auditability.  
2. **Add a defensive schema check** – prepend the script with a small HiveQL block that validates the presence and data types of required columns (`eid`, `commercialoffer`, `month`) before creating the view, failing fast with a clear error message.  

---