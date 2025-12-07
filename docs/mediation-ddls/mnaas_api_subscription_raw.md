**File:** `mediation-ddls\mnaas_api_subscription_raw.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive external table **`mnaas.api_subscription_raw`**. The table is a partition‑by‑month, semicolon‑delimited, text‑file representation of raw API‑subscription events ingested from upstream systems. It lives in the `mnaas` database and points to an HDFS location under the Hive warehouse. Down‑stream mediation jobs (e.g., daily activation, KYC, audit pipelines) read from this table to enrich, aggregate, or validate subscription data.

---

## 2. Core Artifact(s)

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `CREATE EXTERNAL TABLE mnaas.api_subscription_raw` | DDL statement | Defines the schema, partitioning, storage format, and location for raw API subscription records. |
| Table columns (`filename`, `transactionid`, `apiname`, `iccid`, `eid`, `tcl_secs_id`, `status`, `transactiontimestamp`, `inserttimestamp`) | Data definition | Capture the raw payload fields required by downstream mediation logic. |
| Partition column `partition_month` (string) | Partitioning | Enables efficient pruning for month‑level processing (e.g., daily/weekly jobs). |
| Table properties (e.g., `external.table.purge`, `DO_NOT_UPDATE_STATS`) | Hive/Impala metadata | Control lifecycle (purge on DROP) and statistics handling. |

*No procedural code (functions, procedures) is present; the file’s sole purpose is schema creation.*

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | - Raw CSV‑like files (semicolon delimited) placed in HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/api_subscription_raw`.<br>- Partition value supplied via directory naming convention (e.g., `partition_month=202312`). |
| **Outputs** | - Hive/Impala table metadata registered in the Metastore.<br>- Logical view of the underlying HDFS files for SQL queries. |
| **Side Effects** | - Registers an external table; dropping it will **purge** the underlying HDFS files because of `external.table.purge='true'`.<br>- Updates Impala catalog events (catalogServiceId, version). |
| **Assumptions** | - HDFS namespace `NN-HA1` is reachable from Hive/Impala nodes.<br>- Files conform to the declared column order and use `;` as delimiter.<br>- Partition directories exist before data lands (or are created by downstream ingestion jobs).<br>- Hive Metastore and Impala catalog are synchronized. |

---

## 4. Integration Points (How This Table Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **Ingestion pipelines** (e.g., SFTP/FTP drop, Kafka → HDFS writers) | Write raw subscription files into the table’s HDFS location, respecting the `partition_month` directory layout. |
| **Down‑stream mediation scripts** (e.g., `mnaas_actives_raw_daily.hql`, `mnaas_addon_subs_aggr.hql`) | SELECT from `mnaas.api_subscription_raw` to build enriched activation, KYC, or audit datasets. |
| **Impala/Presto query engines** | Serve ad‑hoc analyst queries or reporting dashboards that need raw subscription data. |
| **Data quality / validation jobs** | Run checks on column formats (e.g., timestamp parsing) and partition completeness. |
| **Retention / purge jobs** | May DROP the table (triggering HDFS purge) or ALTER to drop old partitions. |
| **Metadata tools** (e.g., Apache Atlas, Data Catalog) | Pull table properties for lineage tracking; the `impala.events.*` properties help synchronize catalog events. |

*Because all mediation DDL files share the same `mnaas` database, this table is a sibling of other raw tables (`mnaas_actives_raw_daily`, `mnaas_a2p_p2a_sms_audit_raw`, etc.) and is typically joined on `iccid`/`eid`.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream producers change column order or delimiter. | Queries fail or produce corrupt data. | Implement a schema‑validation step in the ingestion pipeline; keep a versioned schema file and compare before loading. |
| **Missing partitions** – data lands in a month directory that is not registered. | Queries on the expected month return zero rows. | Use an automated “add partitions” script (e.g., `MSCK REPAIR TABLE`) after each load, or enforce partition creation in the producer. |
| **Accidental data loss** – `external.table.purge='true'` causes HDFS files to be deleted on DROP. | Irrecoverable loss of raw subscription logs. | Restrict DROP privileges; require a change‑control ticket; consider setting `external.table.purge='false'` and using a separate archival process. |
| **Permission issues** – Hive/Impala service accounts lack read/write on the HDFS path. | Table creation succeeds but queries return “permission denied”. | Verify ACLs on the target directory; document required HDFS permissions in deployment playbooks. |
| **Stale statistics** – `DO_NOT_UPDATE_STATS='true'` may lead to sub‑optimal query plans. | Longer query runtimes, higher cluster load. | Schedule periodic `ANALYZE TABLE … COMPUTE STATISTICS` or enable automatic stats collection for this table. |
| **Impala catalog desynchronization** – catalog version mismatch after table creation. | Impala queries see “table not found” errors. | Run `INVALIDATE METADATA` or `REFRESH` in Impala after DDL execution; automate via a post‑DDL hook. |

---

## 6. Running / Debugging the Script

| Step | Command | Notes |
|------|---------|-------|
| **Execute DDL** | `hive -f mediation-ddls/mnaas_api_subscription_raw.hql` <br>or <br>`impala-shell -i <impala-host> -f mediation-ddls/mnaas_api_subscription_raw.hql` | Use the appropriate client for the environment (Hive for metastore updates, Impala for catalog sync). |
| **Verify creation** | `SHOW CREATE TABLE mnaas.api_subscription_raw;` | Confirms schema, location, and properties. |
| **Inspect partitions** | `SHOW PARTITIONS mnaas.api_subscription_raw;` | Ensure expected `partition_month` values exist. |
| **Check underlying files** | `hdfs dfs -ls /user/hive/warehouse/mnaas.db/api_subscription_raw/partition_month=202312/` | Validate that files are present and correctly delimited. |
| **Test query** | `SELECT COUNT(*) FROM mnaas.api_subscription_raw WHERE partition_month='202312';` | Quick sanity check that data is readable. |
| **Debug common errors** | - *“Table not found”* → run `INVALIDATE METADATA` in Impala.<br>- *“Permission denied”* → check HDFS ACLs.<br>- *“SerDe parsing error”* → verify delimiter and column count in source files. |

---

## 7. External Configuration / Environment Variables

| Config Item | Usage |
|-------------|-------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` clause; must resolve to the active HDFS namenode in the cluster. |
| Hive/Impala service principals | Required for accessing the HDFS path; not visible in the script but assumed to be configured in the cluster security layer (Kerberos). |
| Optional environment variables (e.g., `HIVE_CONF_DIR`, `IMPALA_HOST`) | Used by the execution wrapper (shell script, Airflow operator) that invokes this DDL. |
| Table‑level properties (`impala.events.*`) | Populated automatically by Impala; no external config needed. |

*If the deployment environment uses a different HDFS namespace, the script must be edited or templated to inject the correct namenode address.*

---

## 8. Suggested TODO / Improvements

1. **Add idempotent creation** – prepend `CREATE EXTERNAL TABLE IF NOT EXISTS` (or `DROP TABLE IF EXISTS` followed by `CREATE`) to make the script safe for repeated runs in CI/CD pipelines.  
2. **Automate partition discovery** – include a post‑creation step such as `MSCK REPAIR TABLE mnaas.api_subscription_raw;` or a Spark/Impala job that adds missing `partition_month` directories automatically after each data load.  

---