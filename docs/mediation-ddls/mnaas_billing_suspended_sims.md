**File:** `mediation-ddls\mnaas_billing_suspended_sims.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates (or replaces) the view `mnaas.billing_suspended_sims`. The view joins the `suspended_sim_list` table with the `month_billing` table on the `month` column, filters out three hard‑coded `tcl_secs_id` values, and removes rows where `commercialoffer` is empty. The resulting view is used downstream by billing‑related ETL jobs to identify suspended SIMs that still have a commercial offer attached.

---

## 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `billing_suspended_sims` | Hive **VIEW** (`mnaas.billing_suspended_sims`) | Provides a filtered, joined representation of suspended SIMs together with the month‑billing context for downstream billing calculations. |
| `suspended_sim_list` | Hive **TABLE** (`mnaas.suspended_sim_list`) | Source table containing SIM identifiers, month, and commercial offer information for SIMs that have been suspended. |
| `month_billing` | Hive **TABLE** (`mnaas.month_billing`) | Source table containing month‑level billing metadata (e.g., month key) used to align suspended SIMs with the correct billing period. |

*No procedural code, functions, or classes are defined in this file.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Hive tables `mnaas.suspended_sim_list` and `mnaas.month_billing` must exist and contain the columns `month`, `tcl_secs_id`, `commercialoffer`, and `sim` (for the former) and `month` (for the latter). |
| **Outputs** | - Hive **VIEW** `mnaas.billing_suspended_sims`. The view does **not** materialize data; it is a logical definition that will be evaluated at query time. |
| **Side‑Effects** | - Overwrites an existing view with the same name (implicit `CREATE OR REPLACE VIEW` semantics in Hive). No data is written to HDFS directly. |
| **Assumptions** | - The Hive metastore is reachable and the `mnaas` database is already created. <br> - The `month` column is of the same datatype in both source tables, enabling an equi‑join. <br> - The three excluded `tcl_secs_id` values are static and known to be safe to filter out. <br> - `commercialoffer` is a non‑null string column; empty string (`''`) is considered “no offer”. |
| **External Services** | - HiveServer2 (or equivalent) for DDL execution. <br> - Underlying Hadoop/HDFS storage for the source tables. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Relationship |
|---------------------|--------------|
| **Downstream billing ETL jobs** (e.g., `mnaas_billing_*` scripts) | These jobs query `billing_suspended_sims` to exclude or adjust charges for suspended SIMs that still have a commercial offer. |
| **Data Quality / Validation scripts** | May reference the view to verify that the exclusion list (`tcl_secs_id NOT IN (…)`) is still appropriate. |
| **Orchestration layer** (Airflow, Oozie, or custom scheduler) | The view‑creation script is typically run once per deployment or as part of a “DDL refresh” DAG before nightly billing pipelines. |
| **Configuration repository** | The hard‑coded exclusion list is currently embedded; other scripts that maintain exclusion lists (e.g., a “blacklist” table) could be aligned to this view. |
| **Metadata catalog** (e.g., Apache Atlas) | The view definition is registered as a lineage node, showing dependencies on `suspended_sim_list` and `month_billing`. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded exclusion IDs** may become outdated, causing unintended inclusion/exclusion of SIMs. | Billing inaccuracies. | Externalize the ID list to a configuration table or property file; add a unit test that validates the list against a master blacklist. |
| **Schema drift** in source tables (e.g., column rename or type change) will break view creation or downstream queries. | Job failures, data loss. | Implement schema validation step before view creation; version‑control DDL changes and run integration tests. |
| **Performance**: Join without partition pruning could cause full table scans on large month‑billing data. | Increased query latency, resource contention. | Ensure both source tables are partitioned by `month`; add `/*+ MAPJOIN */` hint if appropriate, or materialize the view as a table for heavy downstream use. |
| **View staleness**: If underlying tables are refreshed (e.g., new month data) but the view is not re‑registered, downstream jobs may see inconsistent data. | Billing mismatches. | Schedule view recreation after any month‑billing data load, or use a “CREATE OR REPLACE VIEW” step in the nightly pipeline. |
| **Missing permissions** for the executing user on the `mnaas` database or source tables. | Script aborts, pipeline halt. | Verify Hive ACLs; include a pre‑flight permission check in the orchestration job. |

---

## 6. Running / Debugging the Script  

| Step | Command / Action |
|------|------------------|
| **Execute** | `hive -f mediation-ddls/mnaas_billing_suspended_sims.hql` (or via `beeline -u jdbc:hive2://<host>:10000 -f …`). |
| **Orchestrated run** | Include the script as a task in an Airflow DAG using `HiveOperator` or Oozie `shell` action. |
| **Validate creation** | `SHOW CREATE VIEW mnaas.billing_suspended_sims;` |
| **Test query** | `SELECT COUNT(*) FROM mnaas.billing_suspended_sims LIMIT 10;` |
| **Debug** | If the view fails to create, run the SELECT portion alone to isolate errors: <br>`SELECT tcl_secs_id, commercialoffer, sim FROM mnaas.suspended_sim_list s JOIN mnaas.month_billing b ON s.month = b.month WHERE tcl_secs_id NOT IN (41218,41648,37226) AND commercialoffer <> '' LIMIT 10;` |
| **Log inspection** | HiveServer2 logs (`/var/log/hive/hiveserver2.log`) will contain any DDL parsing errors. |

---

## 7. External Configuration / Environment Dependencies  

| Item | Usage |
|------|-------|
| **Hive connection parameters** (`HIVE_HOST`, `HIVE_PORT`, `HIVE_USER`, `HIVE_PASSWORD`) | Required by the execution environment (Airflow, Oozie, CLI) to connect to HiveServer2. |
| **Database `mnaas`** | Must exist; typically created by a separate “create database” DDL script. |
| **Potential config file** (e.g., `billing_exclusions.conf`) | Not currently used but recommended for externalizing the `tcl_secs_id` exclusion list. |
| **Hadoop classpath / Hive libraries** | Needed for the Hive CLI/Beeline to parse and execute the script. |

---

## 8. Suggested Improvements (TODO)

1. **Externalize the exclusion list** – Move the hard‑coded `tcl_secs_id` values into a reference table (`mnaas.suspended_sim_exclusions`) or a properties file, and rewrite the view definition to join against that table. This makes the list maintainable without code changes.

2. **Add partition pruning** – If `month_billing` is large, ensure it is partitioned by `month` and add a `WHERE b.month = '${processing_month}'` clause (parameterized via Hive variables) so that only the relevant month’s data is scanned during view evaluation.

---