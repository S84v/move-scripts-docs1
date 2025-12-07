**File:** `mediation-ddls\traffic_aggr_adhoc_view.hql`  

---

## 1. High‑Level Summary
This script creates a Hive **view** named `mnaas.traffic_aggr_adhoc_view`. The view is a thin projection of the external table `mnaas.traffic_aggr_adhoc`, exposing a fixed column list (customer, usage, cost, rating, and partition fields) for downstream reporting, analytics, and ad‑hoc queries. The view does **not** modify data; it only registers a logical layer in the Hive metastore that other jobs, dashboards, or BI tools can reference.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `traffic_aggr_adhoc_view` | Hive **VIEW** | Provides a stable, column‑selected façade over `traffic_aggr_adhoc`. |
| `traffic_aggr_adhoc` | External Hive **TABLE** (defined in `traffic_aggr_adhoc.hql`) | Holds raw, partitioned CDR‑derived usage and cost records that the view reads from. |
| `createtab_stmt` (internal variable) | String | Holds the DDL statement; used only for logging/verification in the execution environment. |

*No procedural code, classes, or functions are present – the file is a pure DDL statement.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Existence of the external table `mnaas.traffic_aggr_adhoc` with the exact column names listed.<br>- Hive metastore connection (via default `hive-site.xml` or environment‑provided JDBC URL). |
| **Outputs** | - A Hive view `mnaas.traffic_aggr_adhoc_view` registered in the metastore.<br>- No physical data files are created. |
| **Side‑Effects** | - Overwrites an existing view with the same name (no `IF NOT EXISTS` guard).<br>- Updates Hive metastore metadata. |
| **Assumptions** | - The external table’s schema matches the column list (order does not matter, but names must exist).<br>- Partitioning on `partition_date` is present and up‑to‑date.<br>- The Hive execution environment has sufficient permissions to CREATE VIEW in database `mnaas`. |
| **External Services** | - HiveServer2 / Metastore (accessed via JDBC/Thrift).<br>- Underlying storage (e.g., HDFS, S3) where `traffic_aggr_adhoc` data resides. |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **`traffic_aggr_adhoc.hql`** | Defines the external table that this view reads. Must be executed **before** this script. |
| **Downstream reporting jobs** (e.g., daily/weekly BI extracts, Spark/Presto queries) | Reference `mnaas.traffic_aggr_adhoc_view` instead of the raw table to obtain a stable column set. |
| **Data ingestion pipelines** (CDR loaders) | Populate `traffic_aggr_adhoc`; any schema change there will affect this view. |
| **Orchestration tools** (Airflow, Oozie, Control-M) | Typically include a task: `hive -f traffic_aggr_adhoc_view.hql` after the table‑creation task. |
| **Version‑control / CI** | The DDL file lives in the `mediation-ddls` repo; CI pipelines validate that the view can be created against a test Hive instance. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – underlying table adds/removes columns used in the view. | View creation fails or returns NULL for missing columns. | Add a CI test that runs `SHOW CREATE VIEW` and validates column existence against the current table schema. |
| **View overwrite** – accidental re‑creation drops dependent objects (e.g., other views referencing it). | Down‑stream jobs may break at runtime. | Use `CREATE OR REPLACE VIEW` (if Hive version supports) or guard with `IF NOT EXISTS` and version the view name (e.g., `traffic_aggr_adhoc_view_v2025_12`). |
| **Permission issues** – user lacks `CREATE VIEW` rights. | Deployment job aborts. | Ensure the execution role is granted `CREATE` on database `mnaas`. Document required Hive ACLs. |
| **Stale data** – view reflects partitions that are not yet loaded. | Reports show incomplete usage. | Coordinate view creation after the nightly partition load; optionally add a `REFRESH` step in the orchestration. |
| **Performance** – view does not prune partitions because `partition_date` is not used in queries. | Full table scans, high latency. | Encourage downstream queries to filter on `partition_date`; consider adding it to the view’s `WHERE` clause if appropriate. |

---

## 6. Running & Debugging the Script  

1. **Standard execution (via Hive CLI or Beeline)**  
   ```bash
   hive -f mediation-ddls/traffic_aggr_adhoc_view.hql
   # or
   beeline -u jdbc:hive2://<host>:10000/default -f mediation-ddls/traffic_aggr_adhoc_view.hql
   ```

2. **Validate creation**  
   ```sql
   SHOW CREATE VIEW mnaas.traffic_aggr_adhoc_view;
   DESCRIBE FORMATTED mnaas.traffic_aggr_adhoc_view;
   ```

3. **Test data access** (quick sanity check)  
   ```sql
   SELECT COUNT(*) FROM mnaas.traffic_aggr_adhoc_view LIMIT 10;
   ```

4. **Debugging failures**  
   - **Error: Table not found** – Verify that `traffic_aggr_adhoc` exists (`SHOW TABLES LIKE 'traffic_aggr_adhoc';`).  
   - **Error: Column not found** – Compare column list in the view DDL with `DESCRIBE mnaas.traffic_aggr_adhoc;`.  
   - **Permission denied** – Check Hive ACLs (`SHOW GRANT USER <user> ON DATABASE mnaas;`).  

5. **Logging** – The orchestration platform should capture Hive stdout/stderr. Look for the `createtab_stmt` line in logs to confirm the exact DDL executed.

---

## 7. External Configuration & Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_CONF_DIR` (or default `/etc/hive/conf`) | Points to Hive configuration (metastore URI, authentication). |
| `HADOOP_USER_NAME` (if Kerberos is not used) | Determines the OS user under which Hive accesses HDFS/S3. |
| No explicit script‑level parameters – the DDL is static. |

If the environment uses a **parameterized Hive execution wrapper** (e.g., `${HIVE_OPTS}`), those flags will affect session settings (e.g., `hive.exec.dynamic.partition=true`).

---

## 8. Suggested Improvements (TODO)

1. **Make the view creation idempotent** – replace the raw `CREATE VIEW` with `CREATE OR REPLACE VIEW` (Hive 2.1+), or add a pre‑check:
   ```sql
   DROP VIEW IF EXISTS mnaas.traffic_aggr_adhoc_view;
   CREATE VIEW ...
   ```

2. **Add documentation comment block** at the top of the file describing purpose, owner, and version, e.g.:
   ```sql
   -- ------------------------------------------------------------
   -- View: traffic_aggr_adhoc_view
   -- Owner: Data Engineering – Mediation Team
   -- Created: 2025-12-04
   -- Description: Stable façade for ad‑hoc traffic usage reporting.
   -- ------------------------------------------------------------
   ```

These changes improve maintainability and reduce accidental deployment failures.