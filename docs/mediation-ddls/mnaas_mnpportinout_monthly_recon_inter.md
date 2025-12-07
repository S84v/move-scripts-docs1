**File:** `mediation-ddls\mnaas_mnpportinout_monthly_recon_inter.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external** table named `mnaas.mnpportinout_monthly_recon_inter`. The table is a lightweight manifest that stores, per processed file, the file name and the number of lines it contained. It is intended to be populated by downstream batch jobs that generate monthly Mobile Number Portability (MNP) port‑in/port‑out reconciliation data. Because the table is external and marked with `external.table.purge='true'`, Hive will not retain data after the underlying HDFS directory is deleted, allowing the reconciliation pipeline to clean up intermediate files safely.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.mnpportinout_monthly_recon_inter` | Hive **external** table | Holds a two‑column manifest (`filename`, `fileline_count`) for each monthly reconciliation file that has been produced. |
| `createtab_stmt` | Variable (used by the build system) | Holds the full `CREATE EXTERNAL TABLE` statement; the script is typically rendered by a templating engine that injects it into a Hive execution context. |
| `LazySimpleSerDe` | SerDe | Parses semi‑colon (`;`) delimited text files. |
| HDFS location `hdfs://NN-HA1/.../mnpportinout_monthly_recon_inter` | Storage | Physical directory where the manifest files are written/read. |

*No procedural code (functions, classes) is present – the file is pure DDL.*

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - The script itself (DDL).<br>- Hive metastore connection (configured via Hive client).<br>- HDFS namenode address (`NN-HA1`). |
| **Outputs** | - A Hive external table definition stored in the metastore.<br>- An HDFS directory (created automatically if missing) that will hold the manifest files. |
| **Side Effects** | - May create the HDFS directory with default permissions (inherits from the parent `mnaas.db` directory).<br>- Registers table metadata (`TBLPROPERTIES`, SerDe config) in the Hive metastore. |
| **Assumptions** | - Hive version supports `external.table.purge` (Hive 2.3+).<br>- The HDFS cluster is reachable and the `NN-HA1` namenode alias resolves correctly.<br>- The `mnaas` database already exists.<br>- No existing table with the same name (or `DROP TABLE IF EXISTS` is handled upstream). |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **Monthly MNP reconciliation jobs** (e.g., `mnpportinout_monthly_recon_job.py` or Spark/MapReduce jobs) | Write a line per processed file to the external table’s HDFS location, using the same `;` delimiter. |
| **Downstream aggregation scripts** (e.g., `mnpportinout_monthly_summary.hql`) | Read from `mnaas.mnpportinout_monthly_recon_inter` to verify completeness of the monthly batch before loading into final fact tables. |
| **Cleanup/orchestration workflows** (Oozie, Airflow, or custom shell scripts) | May delete the HDFS directory after the reconciliation window closes; because `external.table.purge='true'`, Hive automatically drops the table metadata when the directory is removed. |
| **Metadata management tooling** (e.g., schema‑registry CI pipeline) | Validates that the DDL conforms to naming conventions and that the table properties match the environment (production vs. dev). |
| **Security / ACL enforcement** | Relies on HDFS ACLs and Hive metastore permissions; upstream jobs must have write access to the directory and SELECT permission on the table. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale or orphaned HDFS files** – if a downstream job fails, the manifest may contain entries for files that never materialized. | Inaccurate reconciliation status, downstream jobs may abort. | Add a validation step that checks file existence vs. `fileline_count` before downstream consumption. |
| **Permission drift** – changes to HDFS ACLs could prevent writers from appending to the directory. | Job failures, data loss. | Enforce a periodic ACL audit; codify required permissions in the deployment playbook. |
| **Schema drift** – if downstream jobs change delimiter or column order, the SerDe will mis‑parse data. | Corrupted manifest, silent data quality issues. | Version the table (e.g., `mnpportinout_monthly_recon_inter_v2`) and deprecate old versions via CI checks. |
| **Accidental table drop** – external tables can be dropped manually, losing the manifest definition. | Subsequent jobs cannot locate the directory via Hive. | Enable Hive metastore ACLs; consider adding a `DROP TABLE IF EXISTS` guard in the CI‑generated DDL. |
| **HDFS storage bloat** – if `external.table.purge` is disabled or not honored, old files accumulate. | Disk pressure, increased GC in NameNode. | Verify the property is active; schedule a periodic cleanup job that removes directories older than the retention window. |

---

## 6. Running / Debugging the Script  

1. **Standard execution** (via Hive CLI or Beeline):  
   ```bash
   hive -f mediation-ddls/mnaas_mnpportinout_monthly_recon_inter.hql
   # or
   beeline -u jdbc:hive2://<hive-host>:10000 -f mediation-ddls/mnaas_mnpportinout_monthly_recon_inter.hql
   ```

2. **Verification steps** after execution:  
   ```sql
   SHOW CREATE TABLE mnaas.mnpportinout_monthly_recon_inter;
   DESCRIBE FORMATTED mnaas.mnpportinout_monthly_recon_inter;
   ```

3. **Debugging common issues**  
   - *“Table already exists”* → Ensure the CI pipeline runs `DROP TABLE IF EXISTS` first, or use `CREATE EXTERNAL TABLE IF NOT EXISTS`.  
   - *“Path does not exist”* → Verify that the HDFS namenode alias `NN-HA1` resolves and that the parent directory `/user/hive/warehouse/mnaas.db/` is present.  
   - *“SerDe parsing error”* → Check a sample file in the location with `hdfs dfs -cat <path>/part-00000` to confirm the `;` delimiter and line endings.

4. **Log collection**  
   - Hive logs (`/tmp/hive.log` or the Yarn application logs) will contain the DDL execution trace.  
   - HDFS audit logs can confirm directory creation.

---

## 7. External Configuration & Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `hive-site.xml` | Provides Hive metastore URI, authentication method, and default warehouse location. |
| `HADOOP_CONF_DIR` | Supplies the namenode address (`NN-HA1`) and HDFS client settings. |
| `HIVE_EXECUTION_ENGINE` (optional) | Determines whether the DDL runs in MR or Tez; does not affect external table creation but may impact performance of subsequent INSERT/SELECT jobs. |
| `TABLE_PURGE_ENABLED` (custom flag in some CI pipelines) | May be used to toggle `external.table.purge='true'` for non‑production environments. |
| `DB_NAME` (templating variable) | In many pipelines the database name (`mnaas`) is injected; ensure the variable resolves correctly. |

If the script is rendered by a templating engine (e.g., Jinja, Velocity), the variable `createtab_stmt` is substituted with the full DDL before execution.

---

## 8. Suggested Improvements (TODO)

1. **Add Idempotent Guard** – prepend the script with `DROP TABLE IF EXISTS mnaas.mnpportinout_monthly_recon_inter;` to make deployments repeatable without manual cleanup.  
2. **Introduce Table Versioning** – create the table with a version suffix (e.g., `mnpportinout_monthly_recon_inter_v1`) and store the version in a central schema registry. This protects downstream jobs from silent breakage when the manifest schema evolves.  

---