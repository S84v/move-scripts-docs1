**File:** `mediation-ddls\mnaas_mnp_portinout_details_raw.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the managed, transactional Parquet table `mnaas.mnp_portinout_details_raw`. The table stores raw Mobile Number Portability (MNP) “port‑in/out” transaction records as they are ingested from upstream sources (e.g., daily SFTP drops, streaming feeds). It is the canonical raw landing zone for MNP data that downstream mediation jobs (e.g., `mnp_porters_report.hql`, usage‑product tables) read, transform, and aggregate.

---

## 2. Core Object(s)

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.mnp_portinout_details_raw` | Hive/Impala table (managed, transactional, Parquet) | Holds raw MNP transaction rows with fields such as `transactionid`, `msisdn`, `portstartdate`, `porttype`, etc. Provides a stable schema for downstream ETL jobs. |
| `record_insertime` | Column (timestamp) | Auto‑populated by Hive on insert; used for audit / incremental processing. |

*No procedural code (functions, classes) is present in this file; the script’s sole purpose is schema definition.*

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Description |
|--------|-------------|
| **Inputs** | Implicit – the script expects the Hive/Impala environment to be reachable and the HDFS warehouse path to be writable. No external data files are read at creation time. |
| **Outputs** | Creation of a Hive metastore entry and a corresponding directory on HDFS (`hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/mnp_portinout_details_raw`). Subsequent INSERT/LOAD jobs will write Parquet files into this location. |
| **Side Effects** | - Registers the table with Impala catalog (via `impala.events.*` properties). <br>- Enables ACID “insert‑only” transactional semantics. |
| **Assumptions** | - Hive ≥ 2.0 with ACID support enabled. <br>- Impala is configured to read Hive transactional tables. <br>- HDFS namenode alias `NN-HA1` resolves correctly and the `warehouse` directory is accessible to the Hive service account. <br>- No pre‑existing table with the same name (or the operator intends to replace it). |

---

## 4. Integration Points (How This Table Connects to the Rest of the System)

| Down‑stream Component | Connection Detail |
|-----------------------|-------------------|
| `mnaas_mnp_porters_report.hql` (previously processed) | Likely reads from `mnp_portinout_details_raw` to compute aggregated port‑in/out statistics per sponsor/operator. |
| Usage‑product DDLs (`mnaas_gen_usage_product.hql`, `mnaas_gen_sim_product.hql`, etc.) | May join on `msisdn` or `iccid` to enrich usage records with porting information. |
| ETL ingestion jobs (e.g., Oozie/Airflow tasks) | Perform `INSERT INTO mnp_portinout_details_raw SELECT … FROM EXTERNAL_STAGE` after parsing daily CSV/JSON files delivered via SFTP or Kafka. |
| Data quality / validation scripts | Query the table for null/duplicate `transactionid` or inconsistent date ranges. |
| Reporting dashboards (Impala/Presto) | Expose raw port‑in/out data for ad‑hoc analysis. |

*Because the table is transactional and insert‑only, downstream jobs can safely perform incremental inserts without worrying about overwrites.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Table recreation on production** – Running the script repeatedly without `IF NOT EXISTS` will drop/recreate the table, erasing existing data. | Data loss, downstream job failures. | Add `IF NOT EXISTS` clause; keep DDL under version control; use a separate “schema‑migration” process that checks existence before creation. |
| **Schema drift** – Adding/removing columns in later releases without coordinated downstream changes. | Query failures, silent data mismatches. | Maintain a schema change log; enforce backward‑compatible additions (e.g., nullable columns). |
| **HDFS permission issues** – Hive service account lacks write access to the target location. | Table creation fails; pipeline stalls. | Verify HDFS ACLs; include a pre‑flight check script (`hdfs dfs -test -d <path>`). |
| **Transactional overhead** – Insert‑only tables incur extra metadata writes; high volume inserts may degrade performance. | Slower ingestion, possible compaction backlog. | Schedule periodic `ALTER TABLE … COMPACT 'MAJOR'` or enable automatic compaction; monitor `impala`/`hive` compaction metrics. |
| **Impala catalog sync lag** – Impala may not see the new table immediately, causing “Table not found” errors. | Job failures in Impala‑based pipelines. | Run `INVALIDATE METADATA mnaas.mnp_portinout_details_raw;` or `REFRESH` after creation; automate via post‑DDL hook. |

---

## 6. Running / Debugging the Script

1. **Execution**  
   ```bash
   hive -f mediation-ddls/mnaas_mnp_portinout_details_raw.hql
   # or, if using Impala:
   impala-shell -f mediation-ddls/mnaas_mnp_portinout_details_raw.hql
   ```
   *In production pipelines the script is typically invoked by an orchestration tool (Airflow, Oozie) as part of a “schema‑setup” DAG.*

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.mnp_portinout_details_raw;
   DESCRIBE FORMATTED mnaas.mnp_portinout_details_raw;
   SELECT COUNT(*) FROM mnaas.mnp_portinout_details_raw LIMIT 1;
   ```
   Confirm that the table exists, the location points to the expected HDFS path, and the `transactional` property is set.

3. **Debugging Tips**  
   - Check HiveServer2 logs (`/var/log/hive/hiveserver2.log`) for DDL errors.  
   - If Impala cannot see the table, run `INVALIDATE METADATA` or restart the Impala daemon.  
   - Use `hdfs dfs -ls <location>` to ensure the directory was created and has proper permissions.  
   - For ACID metadata issues, inspect the `hive.txn.manager` and `hive.compactor.initiator.on` settings.

---

## 7. External Configurations / Environment Variables

| Config / Variable | Usage in This Script |
|-------------------|----------------------|
| `NN-HA1` (HDFS namenode alias) | Hard‑coded in the `LOCATION` clause; must resolve in the cluster’s DNS or `/etc/hosts`. |
| Hive warehouse root (`hive.metastore.warehouse.dir`) | Implicitly used to construct the full path; the script overrides with an absolute HDFS URI. |
| Impala catalog service ID (`impala.events.catalogServiceId`) & version (`impala.events.catalogVersion`) | Populated automatically by the Hive metastore; not manually set here but recorded for catalog sync. |
| Transactional properties (`transactional`, `transactional_properties`) | Require Hive configuration `hive.txn.manager=org.apache.hadoop.hive.ql.lockmgr.DbTxnManager` and `hive.support.concurrency=true`. Ensure these are enabled cluster‑wide. |

If the environment uses parameter substitution (e.g., `${WAREHOUSE_ROOT}`), the current script does **not** reference them; any change to the HDFS path must be edited directly.

---

## 8. Suggested Improvements (TODO)

1. **Make creation idempotent** – prepend `CREATE TABLE IF NOT EXISTS` and optionally add an `ALTER TABLE` block to add missing columns when evolving the schema.  
2. **Add table comment and column comments** – improves discoverability for downstream analysts and automated documentation tools.  

*Optional*: Consider partitioning by `portstartdate` (or ingestion date) to improve query performance and enable easier data pruning for time‑range reports.  

---