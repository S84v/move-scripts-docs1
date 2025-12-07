**File:** `mediation-ddls\mnaas_billing_active_eids.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view `mnaas.billing_active_eids`. The view joins the `mnaas.active_eid_list` table with the `mnaas.month_billing` table on the `month` column, filters out three specific `tcl_secs_id` values, and excludes rows where `commercialoffer` is empty. The resulting view provides a curated list of active EIDs (equipment identifiers) together with their commercial offer, ready for downstream billing, reporting, and reconciliation processes.

---

## 2. Core Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `billing_active_eids` | Hive **VIEW** | Exposes a filtered, joined dataset of `tcl_secs_id`, `commercialoffer`, and `eid` for the current billing cycle. |
| `active_eid_list` | Hive **TABLE** (populated by `mnaas_active_eid_list.hql`) | Holds the master list of EIDs that are currently active, with a `month` partition column. |
| `month_billing` | Hive **TABLE** (populated by a separate DDL, likely `mnaas_month_billing.hql`) | Contains billing‑cycle metadata (e.g., month identifiers) used to align active EIDs with the correct billing period. |

*No procedural code, functions, or classes are defined in this file; the script is a single DDL statement.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Hive tables: `mnaas.active_eid_list` (must contain columns `tcl_secs_id`, `eid`, `month`).<br>- Hive table: `mnaas.month_billing` (must contain columns `month`, `commercialoffer`). |
| **Outputs** | - Hive **VIEW**: `mnaas.billing_active_eids` (columns `tcl_secs_id`, `commercialoffer`, `eid`). |
| **Side‑Effects** | - Overwrites the view if it already exists (Hive `CREATE VIEW` without `IF NOT EXISTS` will fail if the view exists; in practice the script is usually run with `DROP VIEW IF EXISTS` upstream or via a Hive “replace view” pattern). |
| **Assumptions** | - Both source tables are refreshed **before** this view is (re)created (e.g., nightly ETL jobs that populate `active_eid_list` and `month_billing`).<br>- The `month` column values are consistent across the two tables (same data type, same naming convention).<br>- The three excluded `tcl_secs_id` values are static “black‑list” identifiers that never need to be billed. |
| **External Services** | - Hive Metastore (for DDL execution).<br>- Underlying storage (HDFS, S3, ADLS, etc.) where the source tables reside. |
| **Environment Variables / Config** | None referenced directly in the script. Execution context (e.g., Hive server, Kerberos principal, database connection) is supplied by the job scheduler that runs the script (Airflow, Oozie, etc.). |

---

## 4. Integration Points & Call Graph  

| Connected Component | Relationship |
|---------------------|--------------|
| `mnaas_active_eid_list.hql` | Generates the `active_eid_list` table that feeds this view. |
| `mnaas_month_billing.hql` (or similar) | Generates the `month_billing` table that supplies the `commercialoffer` and `month` columns. |
| Downstream billing jobs (e.g., `mnaas_billing_invoice_generation.hql`) | Consume `billing_active_eids` to calculate invoices, apply discounts, or generate reports. |
| Monitoring / Auditing scripts (e.g., `mnaas_a2p_p2a_sms_audit_raw.hql`) | May reference the view to validate that only allowed EIDs are billed. |
| Scheduler (Airflow DAG, Oozie workflow) | Orchestrates the order: **populate source tables → create/replace view → downstream billing**. |

*Typical execution order (simplified):*  

1. **Refresh source tables** (`active_eid_list`, `month_billing`).  
2. **Run this DDL** to (re)create `billing_active_eids`.  
3. **Trigger downstream processes** that read the view.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table schema drift** (e.g., column rename or type change) | View creation fails; downstream jobs break. | Add schema validation step before view creation (e.g., `DESCRIBE` and assert required columns). |
| **Missing or stale data in source tables** | Billing may be incomplete or inaccurate. | Enforce data‑quality checks (row counts, min/max month) upstream; alert on anomalies. |
| **Hard‑coded blacklist IDs become outdated** | Unintended EIDs may be billed or excluded. | Externalize blacklist to a config table (`billing_blacklist`) and join/filter dynamically. |
| **View recreation race condition** (multiple concurrent runs) | Hive may throw “already exists” errors or produce a partially built view. | Serialize this step in the scheduler (single task instance) or use `CREATE OR REPLACE VIEW` if Hive version supports it. |
| **Permission changes on Hive Metastore** | Script fails with authorization errors. | Verify service account has `CREATE`/`DROP` privileges on the `mnaas` schema; include a pre‑run permission check. |

---

## 6. Running / Debugging the Script  

1. **Manual Execution (CLI)**  
   ```bash
   hive -e "USE mnaas; \
            DROP VIEW IF EXISTS billing_active_eids; \
            CREATE VIEW billing_active_eids AS \
            SELECT tcl_secs_id, commercialoffer, eid \
            FROM active_eid_list a \
            JOIN month_billing b ON a.month = b.month \
            WHERE a.tcl_secs_id NOT IN (41218,41648,37226) \
              AND b.commercialoffer <> '';"
   ```
   *Replace `hive` with `beeline -u <jdbc-url>` if using HiveServer2.*

2. **Within a Scheduler (Airflow example)**  
   ```python
   from airflow.providers.apache.hive.operators.hive import HiveOperator

   create_view = HiveOperator(
       task_id='create_billing_active_eids_view',
       hql="""
           DROP VIEW IF EXISTS mnaas.billing_active_eids;
           CREATE VIEW mnaas.billing_active_eids AS
           SELECT tcl_secs_id, commercialoffer, eid
           FROM mnaas.active_eid_list a
           JOIN mnaas.month_billing b ON a.month = b.month
           WHERE a.tcl_secs_id NOT IN (41218,41648,37226)
             AND b.commercialoffer <> '';
       """,
       hive_cli_conn_id='hive_default',
   )
   ```

3. **Debugging Tips**  
   - Run the **SELECT** part alone to verify row counts and filter logic.  
   - Use `EXPLAIN` on the SELECT to ensure the join uses appropriate partitions.  
   - Check Hive logs (`/var/log/hive/`) for permission or syntax errors.  
   - Verify the view definition after creation: `SHOW CREATE VIEW mnaas.billing_active_eids;`.

---

## 7. External Configuration & Environment Dependencies  

| Item | Usage |
|------|-------|
| Hive Metastore connection (JDBC URL, Kerberos principal) | Provided by the job runner; not hard‑coded in the script. |
| Database/schema name (`mnaas`) | Fixed in the script; must exist and be accessible. |
| Blacklist IDs (`41218,41648,37226`) | Currently hard‑coded; consider moving to a config table or environment variable for easier updates. |
| `commercialoffer` non‑empty filter (`!= ''`) | Assumes empty string denotes “no offer”. If the source uses `NULL` instead, the filter would need adjustment. |

---

## 8. Suggested Improvements (TODO)

1. **Externalize Blacklist** – Create a small reference table `mnaas.billing_eid_blacklist` and replace the hard‑coded `NOT IN (…)` clause with a `LEFT ANTI JOIN`. This makes future additions/removals a data‑ops task rather than a code change.

2. **Add Idempotent View Creation** – Use `CREATE OR REPLACE VIEW` (available in Hive 3.x+) or wrap the DDL with `DROP VIEW IF EXISTS` to avoid manual pre‑drop steps and reduce race conditions.  

---