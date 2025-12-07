**File:** `mediation-ddls\mnaas_ra_traffic_details_summary.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external** table `mnaas.ra_traffic_details_summary` that stores aggregated traffic‑detail metrics (seconds, CDR counts, usage volumes) per `tcl_secs_id`, `proposition`, `country`, `usage_type` and is **partitioned by month**. The table points to a fixed HDFS location under the `mnaas.db` warehouse and is configured for lazy‑serde text input. It is intended for downstream reporting/orchestration jobs that consume pre‑aggregated traffic data across the enterprise.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_traffic_details_summary` | Hive **external** table | Holds month‑partitioned summary rows for traffic metrics (data, voice, SMS, A2P). |
| `createtab_stmt` (variable name only) | DDL string container | Holds the full `CREATE EXTERNAL TABLE …` statement; used by the script runner to execute the DDL. |
| Table **properties** (e.g., `DO_NOT_UPDATE_STATS`, `external.table.purge`) | Metadata | Control statistics collection, automatic purge on drop, and Impala catalog integration. |

*No procedural code (functions, classes) is present in this file; the script is a pure DDL definition.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | - Underlying HDFS directory: `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/ra_traffic_details_summary` <br> - Data files must conform to the column order and types defined (tab‑delimited text, LazySimpleSerDe). |
| **Outputs** | - Hive metastore entry for `mnaas.ra_traffic_details_summary`. <br> - Impala catalog entries (via `impala.events.*` properties). |
| **Side Effects** | - Registers an external table that **does not own** the data; dropping the table will *purge* files because of `external.table.purge='true'`. <br> - Disables automatic statistics updates (`DO_NOT_UPDATE_STATS='true'`). |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves in the execution environment. <br> - The target directory already exists and contains data files matching the schema. <br> - Hive/Impala versions support the listed SerDe and table properties. |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **Data ingestion pipelines** (e.g., `mnaas_ra_traffic_details_*` loaders) | Load raw CDRs, aggregate them, and write the resulting files into the HDFS location defined above. |
| **Reporting / analytics jobs** (e.g., `mnaas_ra_msisdn_wise_usage_report.hql`, `mnaas_ra_file_count_rep_with_reason.hql`) | Query this table to produce monthly usage summaries, KPI dashboards, or billing extracts. |
| **Impala** | The table properties (`impala.events.*`) cause Impala to automatically refresh its catalog; downstream Impala queries rely on this table being present. |
| **Orchestration framework** (Airflow, Oozie, custom scheduler) | Executes this DDL as part of a “DDL deployment” task before any ETL jobs that write to the table. |
| **Metadata governance tools** (e.g., Apache Atlas) | May ingest the table definition for lineage tracking; the external nature and purge flag affect lineage semantics. |

*Because the script only creates the table, any downstream job that expects the table must be scheduled **after** this DDL runs.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed data files** in the external location | Queries return NULLs or fail parsing → inaccurate reports. | Validate file format (delimiter, column count) before loading; add a pre‑run sanity check script. |
| **Partition drift** – new month partitions not created automatically | New data lands in a non‑partitioned directory → full‑table scans. | Use `MSCK REPAIR TABLE` or `ALTER TABLE … ADD PARTITION` as part of the ingestion pipeline. |
| **Statistics disabled** (`DO_NOT_UPDATE_STATS='true'`) | Impala may generate sub‑optimal query plans. | Periodically run `COMPUTE STATS` manually or enable auto‑stats for critical partitions. |
| **Purge on drop** (`external.table.purge='true'`) | Accidental `DROP TABLE` removes raw data permanently. | Protect the table with role‑based permissions; enforce a “soft‑drop” policy (e.g., rename instead of drop). |
| **Hard‑coded HDFS URI** (`NN-HA1`) | Breaks when cluster topology changes or during DR testing. | Externalize the namenode address via a config property or environment variable. |
| **Schema mismatch after upstream changes** | Downstream jobs break. | Version the table (e.g., `ra_traffic_details_summary_v2`) and keep backward‑compatible columns when possible. |

---

## 6. Running / Debugging the Script

1. **Execution (operator)**  
   ```bash
   # Using Hive CLI
   hive -f mediation-ddls/mnaas_ra_traffic_details_summary.hql

   # Or via Beeline (recommended)
   beeline -u "jdbc:hive2://<hive-host>:10000/default" -f mediation-ddls/mnaas_ra_traffic_details_summary.hql
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.ra_traffic_details_summary;
   DESCRIBE FORMATTED mnaas.ra_traffic_details_summary;
   ```

3. **Debugging Tips**  
   - **Metastore errors** – check Hive metastore logs; ensure the `mnaas` database exists.  
   - **Location not accessible** – `hdfs dfs -ls hdfs://NN-HA1/user/hive/warehouse/mnaas.db/ra_traffic_details_summary`.  
   - **Impala catalog sync** – run `INVALIDATE METADATA ra_traffic_details_summary;` in Impala shell after creation.  
   - **Partition visibility** – `SHOW PARTITIONS mnaas.ra_traffic_details_summary;` and compare with HDFS sub‑folders.

4. **Developer iteration**  
   - Edit the DDL locally, run `DROP TABLE IF EXISTS mnaas.ra_traffic_details_summary;` then re‑execute.  
   - Use `EXPLAIN` on a sample query to confirm that Impala picks up the correct partitions and statistics.

---

## 7. External Configuration / Dependencies

| Item | Usage |
|------|-------|
| **HDFS namenode alias** `NN-HA1` | Hard‑coded in the `LOCATION` clause; resolves via the cluster’s DNS or `/etc/hosts`. |
| **Hive/Impala versions** | Must support `LazySimpleSerDe`, `TextInputFormat`, and the listed table properties. |
| **Environment variables** (commonly used in the broader pipeline) | Not referenced directly in this file, but the execution environment typically supplies `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`, and Kerberos credentials for HDFS access. |
| **Metastore connection** | Implicit via the Hive client; requires proper `hive.metastore.uris` configuration. |

If the organization uses a templating system (e.g., Jinja, Velocity) to inject the namenode or database name, verify that the rendered script contains the expected values before execution.

---

## 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend `CREATE EXTERNAL TABLE IF NOT EXISTS` (or wrap with `DROP TABLE IF EXISTS`) to avoid failures on re‑run.  
2. **Automate partition discovery** – add a post‑creation step:  
   ```sql
   MSCK REPAIR TABLE mnaas.ra_traffic_details_summary;
   ```  
   or schedule a periodic `ALTER TABLE … ADD IF NOT EXISTS PARTITION` task to keep the Hive metastore in sync with HDFS.

*Both changes reduce manual intervention and improve reliability of downstream jobs.*