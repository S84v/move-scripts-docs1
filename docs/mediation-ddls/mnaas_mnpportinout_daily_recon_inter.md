**File:** `mediation-ddls\mnaas_mnpportinout_daily_recon_inter.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external** table `mnaas.mnpportinout_daily_recon_inter` that serves as the staging/intermediate layer for the daily Mobile Number Portability (MNP) Port‑In/Port‑Out reconciliation process. The table stores a minimal manifest – the source file name and the number of lines read – for each daily reconciliation file landed in HDFS. Down‑stream mediation jobs (e.g., `mnaas_mnpportinout_daily_recon.hql`, `mnaas_mnpportinout_daily_report.hql`) query this table to drive validation, aggregation, and reporting of MNP traffic.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.mnpportinout_daily_recon_inter` | External Hive table | Holds a two‑column manifest (`filename`, `fileline_count`) for each daily reconciliation file. The table is **external** so the underlying HDFS files are not deleted when the table is dropped (purge enabled). |
| `createtab_stmt` | Variable (used by the build system) | Holds the full `CREATE EXTERNAL TABLE` statement; the build/orchestration framework substitutes this variable into a Hive execution step. |

*No procedural code (UDFs, stored procedures) is defined in this file.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input data** | Files placed in HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/mnpportinout_daily_recon_inter`. Each file must be a delimited text file (`;` delimiter) with a header line that is ignored. |
| **Output** | The Hive metastore entry for the external table and the HDFS directory that stores the raw manifest files. |
| **Side effects** | - Registers the external table in Hive Metastore.<br>- Enables `external.table.purge='true'` – when the table is dropped, the underlying HDFS files are automatically removed (useful for cleanup). |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves correctly in the execution environment.<br>- The Hive warehouse directory (`/user/hive/warehouse/mnaas.db/`) exists and the Hive service user has read/write permissions.<br>- Files arriving in the directory conform to the expected schema (two columns, `;` delimiter, header line). |

---

## 4. Integration Points

| Connected Component | Relationship |
|---------------------|--------------|
| **Up‑stream ingestion jobs** (e.g., `mnaas_mnpportinout_daily_load.hql`) | Write the manifest files into the HDFS location defined above. |
| **Down‑stream reconciliation scripts** (e.g., `mnaas_mnpportinout_daily_recon.hql`, `mnaas_mnpportinout_daily_report.hql`) | Query `mnaas.mnpportinout_daily_recon_inter` to obtain the list of files and line counts for validation and aggregation. |
| **Orchestration layer** (Airflow, Oozie, or custom scheduler) | Executes this DDL as a “create table” task before the daily load/reconciliation DAG runs. |
| **Monitoring / Alerting** | External monitoring may watch the Hive Metastore for the existence of the table or the HDFS directory for file arrival. |
| **Configuration files** | Typically referenced via a central `hive-site.xml` (for `hive.metastore.uris`) and environment variables that define the HDFS namenode (`NN-HA1`). No explicit variable substitution appears in this script, but the build system may inject the `createtab_stmt` variable. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or mis‑named HDFS directory** | Table creation succeeds but subsequent loads fail (file not found). | Pre‑task that validates the directory exists (`hdfs dfs -test -d …`) and creates it if absent. |
| **Insufficient HDFS permissions** | Hive cannot read/write manifest files, causing job failures. | Ensure the Hive service principal/user has `rwx` on the target path; audit ACLs regularly. |
| **Schema drift (extra columns or different delimiter)** | Queries on the table return nulls or raise parsing errors. | Enforce file format validation in the upstream loader; consider adding a Hive `serde` property `serialization.null.format=''`. |
| **Accidental table drop** (purge enabled) | Underlying manifest files are deleted, losing traceability. | Restrict DROP privileges; add a safeguard step in the orchestration that checks a “protected” flag before dropping. |
| **Large number of small files** | HDFS NameNode memory pressure, slower query performance. | Consolidate daily manifests into a single file per day (e.g., using `hdfs dfs -getmerge`) before loading. |
| **Namenode alias change** (`NN-HA1`) | Table points to a stale or unreachable HDFS endpoint. | Store the namenode address in a central config service; update all DDLs via a templating step when the alias changes. |

---

## 6. Execution & Debugging Guide

1. **Run the script**  
   ```bash
   hive -f mediation-ddls/mnaas_mnpportinout_daily_recon_inter.hql
   # or via Beeline
   beeline -u jdbc:hive2://<hive-host>:10000 -f mediation-ddls/mnaas_mnpportinout_daily_recon_inter.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW TABLES LIKE 'mnpportinout_daily_recon_inter';
   DESCRIBE FORMATTED mnaas.mnpportinout_daily_recon_inter;
   ```

3. **Check HDFS location**  
   ```bash
   hdfs dfs -ls hdfs://NN-HA1/user/hive/warehouse/mnaas.db/mnpportinout_daily_recon_inter
   ```

4. **Debug common failures**  
   - *Error: “Path does not exist”* → Verify the namenode alias and directory permissions.  
   - *Error: “SemanticException”* → Ensure Hive metastore is reachable and the `mnaas` database exists.  
   - *No rows returned* → Confirm that upstream jobs have placed files in the directory and that the header line is correctly skipped (`skip.header.line.count='1'`).  

5. **Log inspection**  
   - HiveServer2 logs (`/var/log/hive/hiveserver2.log`).  
   - Application orchestrator logs (Airflow task logs, Oozie job logs).  

---

## 7. External Configuration & Environment Variables

| Config / Variable | Usage |
|-------------------|-------|
| `NN-HA1` | HDFS namenode hostname used in the `LOCATION` clause. Usually supplied via DNS or a cluster‑wide environment variable. |
| `hive.metastore.uris` (in `hive-site.xml`) | Determines where the Hive Metastore service lives; required for table registration. |
| `HIVE_CONF_DIR` | Path to Hive configuration files; must include the above `hive-site.xml`. |
| `HADOOP_CONF_DIR` | Provides HDFS client configuration (core-site.xml, hdfs-site.xml) so the Hive client can resolve `NN-HA1`. |
| `createtab_stmt` (build variable) | Some CI/CD pipelines inject the DDL via this variable; not required for manual execution. |

---

## 8. Suggested Improvements (TODO)

1. **Add partitioning by processing date** – e.g., `PARTITIONED BY (process_date STRING)` – to improve query performance and enable easy purging of old partitions.  
2. **Introduce a table comment** describing the purpose and ownership, and document the expected file naming convention (e.g., `mnpportinout_YYYYMMDD_*.csv`).  

---