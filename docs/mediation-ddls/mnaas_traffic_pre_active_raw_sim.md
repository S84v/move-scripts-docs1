**File:** `mediation-ddls\mnaas_traffic_pre_active_raw_sim.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the managed table **`mnaas.traffic_pre_active_raw_sim`** in the `mnaas` database. The table stores raw, pre‑activation SIM records received from upstream provisioning systems. Each row captures transaction metadata, product and customer identifiers, SIM attributes, carrier information, and the source file name. The table is defined as an *insert‑only transactional* Hive table stored as text files on HDFS under the warehouse path `hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/traffic_pre_active_raw_sim`. Down‑stream mediation jobs (e.g., aggregation, billing, and asset‑mapping scripts listed in the HISTORY) read from this table to enrich, validate, and move data into downstream fact tables.

---

## 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `traffic_pre_active_raw_sim` | Hive Managed Table | Persists raw SIM pre‑activation records; provides a stable schema for downstream ETL jobs. |
| `createtab_stmt` | Variable (script placeholder) | Holds the full `CREATE TABLE` statement; used by the script runner to execute the DDL. |
| `LazySimpleSerDe` with `field.delim=';'` | SerDe | Parses semi‑colon delimited source files into Hive columns. |
| `transactional='true'` / `insert_only` | Table property | Guarantees ACID‑style inserts while preventing updates/deletes (suitable for immutable raw data). |
| HDFS location `hdfs://NN-HA1/.../traffic_pre_active_raw_sim` | Storage | Physical location where Hive stores the text files representing the table. |

*No procedural code (functions, classes) is present; the file is pure DDL.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions
| Category | Details |
|----------|---------|
| **Inputs** | - Source files (semicolon‑delimited) landed in the table’s HDFS location by upstream ingestion jobs (e.g., SFTP drop, Kafka consumer, or batch file mover). |
| **Outputs** | - Hive table `mnaas.traffic_pre_active_raw_sim` populated with rows representing each raw record. |
| **Side‑Effects** | - Creates the table if it does not exist (overwrites if `DROP TABLE` is issued elsewhere). <br>- Registers metadata in Hive Metastore. |
| **Assumptions** | - Hive 2.x+ with ACID support enabled. <br>- HDFS NameNode alias `NN-HA1` resolves in the execution environment. <br>- Upstream processes write files using `;` as field delimiter and `\n` as line delimiter. <br>- No partitioning is required for the ingestion volume (single‑directory layout). |
| **External Services** | - Hive Metastore (metadata service). <br>- HDFS cluster (storage). <br>- Possibly Kerberos or LDAP for authentication (not shown in the script). |

---

## 4. Integration Points with Other Scripts / Components
| Connected Component | Relationship |
|---------------------|--------------|
| `mnaas_traffic_details_raw_daily.hql` | Reads from `traffic_pre_active_raw_sim` (or a downstream staging table) to build daily raw traffic facts. |
| `mnaas_traffic_details_aggr_daily.hql` | Aggregates data that originated in this raw table. |
| `mnaas_tcl_service_asset_mapping_*` scripts | May join on `iccid` / `msisdn` fields to map SIMs to service assets. |
| Ingestion pipelines (SFTP, Kafka, or batch file mover) | Deposit raw files into the HDFS location defined by the table’s `LOCATION`. |
| Monitoring / Alerting tools | Track table size, row count, and ingestion latency; rely on Hive Metastore to confirm table existence. |
| Data quality jobs (e.g., reject‑record scripts) | May read from this table to flag malformed rows before they flow downstream. |

*Because all DDL files share the same `mnaas` database, they collectively form the schema backbone for the mediation layer.*

---

## 5. Operational Risks & Recommended Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – downstream jobs expect a column that is renamed or removed. | Job failures, data loss. | Version‑control DDL; add a migration script that adds new columns with `ALTER TABLE … ADD COLUMNS` rather than dropping/recreating. |
| **Incorrect delimiter** – upstream file uses a different delimiter, causing mis‑parsed rows. | Corrupt data, downstream rejections. | Validate source files with a pre‑load checksum or schema‑validation step; enforce delimiter via ingestion job. |
| **Unbounded table size** – no partitioning leads to performance degradation as data grows. | Slow queries, high GC pressure. | Introduce date‑based partitioning (e.g., `PARTITIONED BY (ingest_date STRING)`) and rewrite ingestion to write into the appropriate partition. |
| **Transactional property misuse** – attempts to UPDATE/DELETE rows (not allowed with `insert_only`). | Job errors. | Document that the table is immutable; any corrections must be performed by inserting corrected rows into a new table and swapping. |
| **HDFS path change** – NameNode alias or warehouse path changes without updating the DDL. | Table becomes inaccessible. | Externalize the base HDFS URI into a config variable (e.g., `${WAREHOUSE_ROOT}`) and reference it via Hive variables (`SET hive.exec.scratchdir=${WAREHOUSE_ROOT}`) when running the script. |
| **Missing Hive Metastore sync** – after table creation, downstream jobs start before Metastore propagation. | “Table not found” errors. | Add a short `SHOW TABLES LIKE 'traffic_pre_active_raw_sim'` check or a retry loop in orchestration workflow. |

---

## 6. Running / Debugging the Script
1. **Execution**  
   ```bash
   hive -f mediation-ddls/mnaas_traffic_pre_active_raw_sim.hql
   ```
   *Or* submit via an orchestration tool (e.g., Oozie, Airflow) that runs Hive actions.

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.traffic_pre_active_raw_sim;
   DESCRIBE FORMATTED mnaas.traffic_pre_active_raw_sim;
   SELECT COUNT(*) FROM mnaas.traffic_pre_active_raw_sim LIMIT 10;
   ```

3. **Common Debug Steps**  
   - Check Hive Metastore logs for DDL parsing errors.  
   - Verify HDFS path exists and is writable: `hdfs dfs -ls /warehouse/tablespace/managed/hive/mnaas.db/traffic_pre_active_raw_sim`.  
   - If the table already exists and you need to replace it, run `DROP TABLE IF EXISTS mnaas.traffic_pre_active_raw_sim;` before re‑executing.  
   - Use `hive --hiveconf hive.exec.orc.default.compress=ZLIB` (or other Hive configs) only if you change storage format later.

4. **Logging**  
   - Hive logs are written to `$HIVE_LOG_DIR` (often `/tmp/hive.log`).  
   - For Airflow/Oozie, inspect the task’s stdout/stderr artifacts.

---

## 7. External Configuration / Environment Dependencies
| Config / Env Variable | Usage |
|-----------------------|-------|
| `NN-HA1` (NameNode alias) | Hard‑coded in the `LOCATION` clause; must resolve in the cluster DNS or `/etc/hosts`. |
| Hive Metastore connection (`hive.metastore.uris`) | Required for DDL registration. |
| `hive.exec.scratchdir` (optional) | If you externalize the warehouse root, you could replace the absolute HDFS URI with a variable. |
| Kerberos principal / keytab (if security enabled) | Needed for Hive and HDFS access; not referenced in the script but required at runtime. |

*No placeholder variables are used in the current file; any change to the storage path should be driven by a configuration parameter to avoid hard‑coding.*

---

## 8. Suggested Improvements (TODO)
1. **Add `IF NOT EXISTS` guard** – Prevent accidental failures when the script is re‑run:  
   ```sql
   CREATE TABLE IF NOT EXISTS mnaas.traffic_pre_active_raw_sim ( ... );
   ```

2. **Introduce date‑based partitioning** to improve query performance and manage data lifecycle:  
   ```sql
   PARTITIONED BY (ingest_date STRING)
   LOCATION 'hdfs://NN-HA1/.../traffic_pre_active_raw_sim';
   ```

   Then adjust upstream ingestion to write files into `.../traffic_pre_active_raw_sim/ingest_date=YYYYMMDD/`.

---