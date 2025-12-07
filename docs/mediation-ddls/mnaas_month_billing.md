**File:** `mediation-ddls\mnaas_month_billing.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala external table **`mnaas.month_billing`**. The table stores month‑level billing window metadata (month identifiers, start/end timestamps and dates) as plain‑text files located in HDFS under `/user/hive/warehouse/mnaas.db/month_billing`. It is part of the “mediation‑ddls” suite that defines the raw data model for downstream billing, reconciliation, and reporting jobs.

---

## 2. Core Definition (Responsibility)

| Element | Description |
|---------|-------------|
| **Table name** | `mnaas.month_billing` – resides in the `mnaas` database. |
| **Columns** | `month` (string), `next_month` (string), `start_with_time` (string), `end_with_time` (string), `start_date` (string), `end_date` (string). All columns are currently defined as `STRING`. |
| **Storage** | External table, **TEXTFILE** format (`LazySimpleSerDe`). |
| **Location** | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/month_billing` – a dedicated HDFS directory. |
| **Table properties** | - `external.table.purge='true'` (drop will delete underlying files). <br> - Impala catalog metadata (`impala.events.*`, `impala.lastComputeStatsTime`). <br> - Audit fields (`last_modified_by`, `last_modified_time`, `transient_lastDdlTime`). |
| **Purpose** | Provides a static reference for month‑boundary data used by billing‑cycle calculations, reconciliation scripts, and any job that needs to map timestamps to a billing month. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | None at execution time – the script is self‑contained. The table will later be populated by a separate data‑load process (e.g., an ETL job that writes CSV/TSV files to the HDFS location). |
| **Outputs** | Creation of the external table metadata in Hive Metastore / Impala catalog. No data is written by this script. |
| **Side Effects** | - Registers a new HDFS path as an external table. <br> - Because `external.table.purge='true'`, a subsequent `DROP TABLE` will delete the files under the location, which is a destructive side effect. |
| **Assumptions** | - HDFS namenode alias `NN-HA1` is reachable from the Hive/Impala service. <br> - The directory `/user/hive/warehouse/mnaas.db/month_billing` either does not exist or is empty; otherwise the table may point to pre‑existing data. <br> - Hive Metastore is running and the `mnaas` database already exists. |

---

## 4. Integration Points (How it Connects to Other Scripts)

| Connected Component | Interaction |
|---------------------|-------------|
| **Data‑load jobs** (e.g., `mnaas_month_billing_load.hql` or Spark/MapReduce jobs) | Write CSV/TSV files into the HDFS location defined above. Those jobs reference the table name `mnaas.month_billing` for validation or downstream joins. |
| **Reconciliation scripts** (e.g., `mnaas_monthly_recon.hql`) | Join on `month`/`start_date` fields to align transaction records with the correct billing window. |
| **Reporting pipelines** (e.g., Tableau/PowerBI connectors) | Query the table via Impala to retrieve month boundaries for UI filters. |
| **DDL orchestration** (e.g., a master `create_all_ddls.sh` script) | This file is invoked together with the other `mediation-ddls\*.hql` files to provision the full schema set before any data loads run. |
| **Metadata audit tools** | The `last_modified_*` properties are read by governance scripts that track schema changes. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental drop → data loss** (because of `external.table.purge='true'`) | Permanent loss of month‑billing files if the table is dropped. | - Enforce a change‑management approval step before any `DROP TABLE`. <br> - Add a protective comment in the script warning operators. |
| **Schema mismatch (all columns are STRING)** | Downstream jobs may need DATE/TIMESTAMP types and perform costly casts. | - Review downstream consumption; if dates are required, change column types to `DATE`/`TIMESTAMP` and back‑fill data. |
| **Incorrect HDFS path or permissions** | Table creation succeeds but queries fail due to inaccessible files. | - Validate HDFS connectivity and directory permissions before running the DDL (e.g., `hdfs dfs -test -d <path>`). |
| **Duplicate table creation** | Running the script multiple times without `IF NOT EXISTS` may error out, breaking automated pipelines. | - Add `IF NOT EXISTS` clause or wrap execution in a try‑catch block in the orchestration script. |
| **Stale metadata** (Impala catalog not refreshed) | Queries return old schema or empty results after a change. | - Issue `INVALIDATE METADATA mnaas.month_billing;` or `REFRESH` after DDL execution. |

---

## 6. Running / Debugging the Script

1. **Typical execution** (via Hive CLI or Ambari/Beeline):  
   ```bash
   hive -f mediation-ddls/mnaas_month_billing.hql
   ```
   Or, for Impala:  
   ```bash
   impala-shell -i <impala-coordinator> -f mediation-ddls/mnaas_month_billing.hql
   ```

2. **Verification steps**  
   ```sql
   SHOW CREATE TABLE mnaas.month_billing;
   DESCRIBE FORMATTED mnaas.month_billing;
   SELECT * FROM mnaas.month_billing LIMIT 5;   -- should return 0 rows initially
   ```

3. **Debugging tips**  
   - If the command fails with “Database does not exist”, create the `mnaas` database first (`CREATE DATABASE IF NOT EXISTS mnaas;`).  
   - Check HDFS path existence and permissions: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/month_billing`.  
   - Look at Hive Metastore logs for “Table already exists” errors; consider adding `IF NOT EXISTS`.  
   - After creation, run `INVALIDATE METADATA` in Impala to ensure the catalog is up‑to‑date.

---

## 7. External Configuration / Environment Variables

| Item | Usage |
|------|-------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` clause. If the cluster’s namenode address changes, the script must be edited or templated. |
| Hive/Impala connection parameters (e.g., `hive.metastore.uris`, `impala.host`) | Not referenced directly in the script but required by the client used to run it. |
| Optional templating system (e.g., Jinja, envsubst) | Not present in the current file; any future parameterisation would need to be added manually. |

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` and explicit `DROP` handling**  
   ```sql
   DROP TABLE IF EXISTS mnaas.month_billing PURGE;
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.month_billing ( ... );
   ```
   This makes the script idempotent for CI/CD pipelines.

2. **Use proper data types** – change `STRING` columns that represent dates/times to `DATE` or `TIMESTAMP`. Example:  
   ```sql
   `start_date` DATE,
   `end_date`   DATE,
   `start_with_time` TIMESTAMP,
   `end_with_time`   TIMESTAMP
   ```
   This reduces casting overhead in downstream jobs and improves query performance.

---