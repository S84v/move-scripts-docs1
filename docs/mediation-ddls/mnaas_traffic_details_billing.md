**File:** `mediation-ddls\mnaas_traffic_details_billing.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive/Impala table** named `traffic_details_billing` in the `mnaas` database. The table maps raw billing‑related CDR (Call Detail Record) fields to a flat, delimited file stored under a fixed HDFS location. By declaring the table as *external* with `external.table.purge='true'`, the metadata lives in the Hive Metastore while the underlying files remain on HDFS and are automatically removed when the table is dropped. The table is later referenced by downstream aggregation, reporting, and reconciliation jobs (e.g., `mnaas_traffic_details_aggr_daily.hql`, `mnaas_ra_traffic_details_summary.hql`).

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `traffic_details_billing` | External Hive table | Provides a schema‑on‑read view of billing‑level CDR files stored in HDFS. |
| `createtab_stmt` | DDL statement (variable name used by the build framework) | Holds the full `CREATE EXTERNAL TABLE …` statement that the deployment engine executes. |
| `ROW FORMAT SERDE` | SerDe definition | Uses `LazySimpleSerDe` to parse delimited text (default field delimiter – usually `\001`). |
| `INPUTFORMAT / OUTPUTFORMAT` | Hadoop I/O classes | Reads plain text files (`TextInputFormat`) and writes using Hive’s ignore‑key text output format. |
| `TBLPROPERTIES` | Table metadata | Enables external‑table purge, records Impala catalog IDs, and stores statistics timestamps. |

*No procedural code (functions, classes) exists in this file; it is pure DDL.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Raw billing CDR files placed in HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_billing`.<br>- Implicit delimiter (default `\001` unless overridden by session settings). |
| **Outputs** | - Hive/Impala table metadata registered in the Metastore under `mnaas.traffic_details_billing`.<br>- No data is written by this script; it only points to existing files. |
| **Side‑Effects** | - Creates an entry in the Hive Metastore.<br>- Sets `external.table.purge='true'` → dropping the table will delete the underlying HDFS files.<br>- Updates Impala catalog version properties. |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves in the execution environment.<br>- The target directory exists and contains files matching the column order and data types.<br>- No partitioning is required for downstream jobs (they will scan the whole location).<br>- The Hive/Impala services share the same Metastore. |

---

## 4. Integration Points  

| Connected Component | How It Links |
|---------------------|--------------|
| **Downstream aggregation scripts** (`mnaas_traffic_details_aggr_daily.hql`, `mnaas_ra_traffic_details_summary.hql`) | Query `traffic_details_billing` to compute daily usage, billing totals, and summary statistics. |
| **Ingestion pipelines** (e.g., nightly Spark/MapReduce jobs) | Write raw CDR files into the HDFS location before this DDL is executed; the table provides a read‑only view for analytics. |
| **Data quality / validation jobs** | May read the table to validate column formats, nullability, and record counts against source systems. |
| **Impala UI / BI tools** | Users query `mnaas.traffic_details_billing` directly for ad‑hoc analysis. |
| **Metastore backup / migration scripts** | Rely on the table definition being present in the Metastore; the external flag ensures data is not duplicated during backups. |

*Note:* The naming convention (`traffic_details_*`) suggests a family of tables that share the same raw source but are filtered or transformed for specific purposes (e.g., raw daily, aggregated daily, billing). This DDL is the “billing” slice.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – source CDR files change column order or data type. | Queries fail, data corruption. | Implement a schema‑validation step (e.g., Spark job that checks the first N rows against the DDL) before loading new files. |
| **Accidental table drop** – `external.table.purge='true'` will delete the raw files. | Permanent loss of raw billing data. | Restrict DROP privileges; enforce a change‑control process; add a pre‑drop safeguard script that backs up the HDFS directory. |
| **Missing HDFS path** – directory not created or permissions insufficient. | Table creation succeeds but queries return empty set or error. | Include a provisioning script that `hdfs dfs -mkdir -p` the path with proper ACLs; verify with `hdfs dfs -test -d`. |
| **Improper delimiter** – default delimiter does not match source file. | Mis‑parsed columns, data shift. | Explicitly set `FIELDS TERMINATED BY` in the DDL or enforce a standard delimiter upstream. |
| **Performance degradation** – full‑scan on a non‑partitioned table. | Long query runtimes, high cluster load. | Consider adding a `PARTITIONED BY (partition_date STRING)` clause and reorganizing files into date‑based subfolders. |
| **Impala catalog out‑of‑sync** – catalog version mismatch after DDL changes. | Stale metadata, query failures. | Run `INVALIDATE METADATA traffic_details_billing;` after DDL execution in Impala. |

---

## 6. Running / Debugging the Script  

1. **Execution**  
   ```bash
   # From a Hive/Impala client machine
   hive -f mediation-ddls/mnaas_traffic_details_billing.hql
   # or, for Impala
   impala-shell -i <impala-coordinator> -f mediation-ddls/mnaas_traffic_details_billing.hql
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.traffic_details_billing;
   DESCRIBE FORMATTED mnaas.traffic_details_billing;
   SELECT COUNT(*) FROM mnaas.traffic_details_billing LIMIT 10;
   ```

3. **Debugging Tips**  
   - If the table appears but returns zero rows, check the HDFS directory: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/traffic_details_billing`.  
   - Use `hdfs dfs -cat` on a sample file to confirm delimiter and column order.  
   - In Impala, run `INVALIDATE METADATA mnaas.traffic_details_billing;` then re‑query.  
   - Review Metastore logs (`/var/log/hive/hive-metastore.log`) for DDL parsing errors.  

---

## 7. External Configuration & Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `NN-HA1` (namenode alias) | Resolved by the Hadoop client to locate the HDFS cluster. Typically defined in `core-site.xml` or via DNS. |
| Hive/Impala connection strings (`hive.metastore.uris`, `impala.catalog.service.id`) | Not referenced directly in the file but required for the client that runs the DDL. |
| `HIVE_CONF_DIR` / `IMPALA_HOME` | Determines which `hive-site.xml` / `impala-shell` configuration is used when executing the script. |
| Optional: `HADOOP_USER_NAME` | Determines the HDFS user that must have write permission on the target directory. |

If the deployment framework injects the DDL via a variable (`createtab_stmt`), ensure the surrounding script sources the correct environment (e.g., `source /opt/mnaas/conf/env.sh`).

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Introduce `PARTITIONED BY (partition_date STRING)` and reorganize incoming files into `.../traffic_details_billing/partition_date=YYYYMMDD/` to enable predicate push‑down and reduce scan size for daily jobs.  
2. **Explicit Delimiter & Column Comments** – Define `FIELDS TERMINATED BY '\t'` (or the actual delimiter) and add `COMMENT` clauses for each column to improve data‑dictionary generation and downstream developer understanding.  

---