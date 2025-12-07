**File:** `mediation-ddls\mnaas_tolling_sim_list.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates the **external** table `mnaas.tolling_sim_list`. The table stores SIM‑related tolling metadata (SIM identifier, TCL security ID, organization name, commercial offer, and EID) and is **partitioned by month**. Data lives in an HDFS directory under the Hive warehouse and is managed outside the Hive metastore (i.e., Hive does not own the files). The table is a foundational data source for downstream tolling, billing, and reporting jobs that join SIM information with usage or charging records.

---

## 2. Core Artifact(s)

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `CREATE EXTERNAL TABLE mnaas.tolling_sim_list` | DDL statement | Defines schema, partitioning, storage format, and HDFS location for the tolling SIM list. |
| Table properties (`DO_NOT_UPDATE_STATS`, `external.table.purge`, etc.) | Metastore metadata | Controls statistics handling, automatic data purge on DROP, and Impala catalog integration. |

*No procedural code (functions, procedures, or scripts) is present in this file; its sole purpose is schema definition.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | - Implicit: Hive/Impala metastore connection.<br>- Implicit: HDFS namenode address (derived from Hive configuration). |
| **Outputs** | - New external table metadata stored in the Hive metastore.<br>- No data files are created; the table points to an existing (or future) HDFS directory: `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/tolling_sim_list`. |
| **Side Effects** | - Registers a new table in the metastore (visible to Hive, Impala, Spark, etc.).<br>- If the table already exists, the script will fail (no `IF NOT EXISTS`). |
| **Assumptions** | - HDFS path `hdfs://NN-HA1/.../tolling_sim_list` is reachable and has appropriate read/write ACLs for the Hive service user.<br>- The Hive metastore is reachable and the `mnaas` database already exists.<br>- Downstream pipelines will populate the partition directories (`month=YYYYMM`) with delimited text files matching the defined column order. |

---

## 4. Integration Points & Connectivity

| Connected Component | How the Table Is Used |
|---------------------|-----------------------|
| **Ingestion pipelines** (e.g., nightly ETL jobs, streaming Spark jobs) | Write CSV/TSV files into `.../tolling_sim_list/month=YYYYMM/`; then run `ALTER TABLE mnaas.tolling_sim_list ADD IF NOT EXISTS PARTITION (month='YYYYMM') LOCATION '.../month=YYYYMM'`. |
| **Reporting / analytics jobs** (Hive/Impala/Presto queries) | Join `tolling_sim_list` with usage or charging tables (`tolling_usage`, `billing_events`, etc.) to enrich tolling calculations. |
| **Data quality / validation scripts** | Query `SELECT COUNT(*) FROM mnaas.tolling_sim_list WHERE sim IS NULL` to detect malformed rows. |
| **Metadata tools** (e.g., Apache Atlas, DataHub) | Harvest table definition and properties for lineage tracking. |
| **Other DDL scripts in `mediation-ddls`** | Follow the same naming convention (`mnaas_<entity>_...`) and are typically executed together during a schema rollout or migration. |

*Because the table is external, any process that removes or archives HDFS files will affect downstream consumers without Hive‑level protection.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing HDFS directory or permission errors** | Table creation succeeds but queries fail (data not readable). | Verify the directory exists and Hive user (`hive` or service principal) has `rwx` on the path before running the script. |
| **Stale partitions** (old month folders left behind) | Queries may scan unnecessary data, inflating runtime and costing resources. | Implement a periodic partition‑prune job that drops partitions older than a retention window. |
| **Accidental data loss** (because `external.table.purge='true'`) | Dropping the table will delete underlying files. | Restrict DROP privileges; add a safeguard script that disables purge before any DROP operation. |
| **Schema drift** (source files not matching column order/types) | Query failures or silent data corruption. | Enforce schema validation in the ingestion pipeline (e.g., Spark schema enforcement) and add a unit test that reads a sample file and validates column count/types. |
| **Duplicate table creation** (script re‑run without `IF NOT EXISTS`) | Job failure, blocking downstream pipelines. | Add `IF NOT EXISTS` to the CREATE statement or wrap execution in a try/catch block in the orchestration layer. |

---

## 6. Running / Debugging the Script

1. **Typical execution (Hive CLI)**  
   ```bash
   hive -f mediation-ddls/mnaas_tolling_sim_list.hql
   ```

2. **Impala / Beeline**  
   ```bash
   beeline -u "jdbc:hive2://<impala-host>:21050/default;auth=noSasl" -f mediation-ddls/mnaas_tolling_sim_list.hql
   ```

3. **Verification steps**  
   ```sql
   -- Verify table exists
   SHOW TABLES LIKE 'tolling_sim_list';

   -- Inspect schema
   DESCRIBE FORMATTED mnaas.tolling_sim_list;

   -- Check location
   DESCRIBE EXTENDED mnaas.tolling_sim_list;
   ```

4. **Debugging common issues**  
   * **Metastore error** – Check Hive metastore logs; ensure the `mnaas` database exists (`CREATE DATABASE IF NOT EXISTS mnaas;`).  
   * **HDFS path not reachable** – Run `hdfs dfs -ls hdfs://NN-HA1/user/hive/warehouse/mnaas.db/tolling_sim_list`.  
   * **Permission denied** – Verify ACLs: `hdfs dfs -getfacl <path>` and adjust with `hdfs dfs -setfacl`.  

5. **Automation**  
   Include the script in a CI/CD pipeline (e.g., Jenkins, GitLab CI) that runs `hive -f` against a test metastore before promotion to production.

---

## 7. External Configuration & Environment Variables

| Config / Variable | Usage |
|-------------------|-------|
| `hive.metastore.uris` | Hive client uses this to connect to the metastore where the table definition is stored. |
| `fs.defaultFS` (or `fs.default.name`) | Determines the default HDFS namenode; the script references `hdfs://NN-HA1/...`. |
| `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Location of `hive-site.xml` and `impala-site.xml` that contain the above properties. |
| `WAREHOUSE_DIR` (optional) | If the environment overrides the default warehouse location, the absolute path in the script must still be valid. |
| `HADOOP_USER_NAME` (or Kerberos principal) | Determines the effective HDFS user for permission checks. |

*The script itself does not read any environment variables; it relies on the Hive/Impala client configuration.*

---

## 8. Suggested TODO / Improvements

1. **Make the DDL idempotent** – Add `IF NOT EXISTS` to avoid failures on re‑run:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.tolling_sim_list ( ... )
   ```

2. **Add a comment block** describing the purpose, data owner, and retention policy, e.g.:  
   ```sql
   COMMENT 'SIM‑to‑TCL mapping for tolling. Populated nightly by <pipeline>. Retention: 12 months.'
   ```

3. *(Optional)* **Introduce a staging location** and a `LOAD DATA INPATH` step in a separate script to decouple file landing from table definition, improving data‑pipeline reliability.

---