**File:** `mediation-ddls\mnaas_asset_based_service_status_reject.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external** table `mnaas.asset_based_service_status_reject` that stores rejected asset‑based service‑status events. The table is partitioned by `event_month` and points to a fixed HDFS location where raw, delimited text files are landed by upstream mediation jobs. Because it is external, Hive only registers the metadata; the actual data files are managed outside Hive (e.g., by an SFTP ingest or a streaming pipeline). Down‑stream analytics, reporting, or remediation processes query this table to understand why certain service‑status updates were rejected.

---

## 2. Core Definition (Responsibility)

| Element | Responsibility |
|---------|-----------------|
| **CREATE EXTERNAL TABLE** | Registers a Hive metastore entry without moving or copying data. |
| **Columns (`sourceid` … `reject_reason`)** | Capture raw event attributes as strings (source system ID, timestamps, identifiers, status, reject reason, etc.). |
| **PARTITIONED BY (`event_month`)** | Enables efficient pruning for month‑level queries and aligns with daily/weekly ingestion partitions. |
| **ROW FORMAT SERDE** (`LazySimpleSerDe`) | Parses delimited text (default delimiter is `\001`). |
| **INPUTFORMAT / OUTPUTFORMAT** | Uses Hadoop’s `TextInputFormat` and `HiveIgnoreKeyTextOutputFormat` for plain‑text files. |
| **LOCATION** | Points to `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/asset_based_service_status_reject`. All files placed here become visible to the table. |
| **TBLPROPERTIES** | `OBJCAPABILITIES='EXTREAD,EXTWRITE'` signals that external reads/writes are allowed; `transient_lastDdlTime` records the DDL timestamp. |

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | - Raw text files dropped into the HDFS location (by upstream mediation jobs, SFTP, or streaming ingest). <br> - Partition value (`event_month`) derived from file naming or directory structure (e.g., `event_month=202312`). |
| **Outputs** | - Hive metastore entry for `asset_based_service_status_reject`. <br> - Queryable virtual table; no physical data movement. |
| **Side Effects** | - May overwrite an existing table definition if the script is re‑run (Hive will replace the metadata). <br> - Alters Hive’s namespace (`mnaas` database). |
| **Assumptions** | - HDFS path exists and is writable by the Hive service account. <br> - Incoming files conform to the column order and delimiter expected by `LazySimpleSerDe`. <br> - `event_month` partition directories are created before data lands (or `MSCK REPAIR TABLE` is run later). <br> - Hive version supports external tables with the specified SerDe. |

---

## 4. Integration Points (How It Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **Upstream Mediation Jobs** (e.g., `mnaas_asset_based_service_status` pipeline) | Write rejected event records to the HDFS location, usually under a partition folder `event_month=YYYYMM`. |
| **Partition Discovery Scripts** (`MSCK REPAIR TABLE` or custom Spark jobs) | After files land, a scheduled job runs `MSCK REPAIR TABLE mnaas.asset_based_service_status_reject;` to add new partitions to the metastore. |
| **Downstream Analytics / Reporting** (BI dashboards, Spark/Presto queries) | Query the table to count rejects, drill‑down by `reject_reason`, or join with the main `asset_based_service_status` table for reconciliation. |
| **Data Quality / Monitoring** (e.g., Airflow DAG, Oozie workflow) | DAG tasks may check row counts, validate that `event_month` partitions are present, and raise alerts if the table is empty for a given period. |
| **Retention / Archival Jobs** | Periodic scripts may move old partition directories to an archive HDFS path and then drop the partition from Hive. |
| **Security / ACLs** | Hive ACLs and HDFS permissions must allow the ingestion service to write files and the analytics service to read them. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream changes column order or adds new fields. | Queries break; data mis‑aligned. | Enforce schema contract via a validation step (e.g., a Spark job that checks column count before landing). |
| **Missing partitions** – files land without the expected `event_month` folder. | Data invisible to Hive; downstream jobs see gaps. | Automate `MSCK REPAIR TABLE` after each ingest or use `ALTER TABLE ADD PARTITION` in the ingestion script. |
| **HDFS path deletion / permission change** – accidental removal of the external directory. | Complete data loss for this table. | Apply HDFS ACLs, enable directory snapshots, and monitor path existence with a health check. |
| **Large number of small files** – ingestion creates many tiny files. | Query performance degradation, NameNode pressure. | Consolidate files via a periodic compaction job (e.g., Spark `coalesce` + overwrite). |
| **Incorrect data types (all strings)** – downstream calculations require proper types. | Additional casting overhead, possible runtime errors. | Consider altering the DDL to use appropriate types (`timestamp`, `bigint`) and add a conversion layer in the ingestion pipeline. |

---

## 6. Running / Debugging the Script

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_asset_based_service_status_reject.hql
   ```
   - Verify success message: `Table created successfully`.

2. **Validate Table Metadata**  
   ```sql
   DESCRIBE FORMATTED mnaas.asset_based_service_status_reject;
   SHOW PARTITIONS mnaas.asset_based_service_status_reject;
   ```

3. **Check Sample Data** (after a partition is added)  
   ```sql
   SELECT * FROM mnaas.asset_based_service_status_reject
   WHERE event_month = '202312' LIMIT 10;
   ```

4. **Debugging Tips**  
   - If the table already exists, drop it first (`DROP TABLE IF EXISTS mnaas.asset_based_service_status_reject;`).  
   - Ensure the Hive metastore is reachable (`hive --service metastore` health).  
   - Look at Hive logs (`/var/log/hive/hive.log`) for SerDe or InputFormat errors.  
   - Verify HDFS permissions: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/asset_based_service_status_reject`.

---

## 7. External Config / Environment Variables

| Variable / Config | Usage |
|-------------------|-------|
| `HIVE_HOME`, `HADOOP_CONF_DIR` | Required for the `hive` CLI to locate configuration files. |
| `NN-HA1` (namenode address) | Embedded in the `LOCATION` URI; may be overridden by a cluster‑wide alias in `core-site.xml`. |
| Hive metastore connection (`javax.jdo.option.ConnectionURL`, etc.) | Determines where the table definition is stored. |
| Optional: `hive.exec.dynamic.partition` / `hive.exec.dynamic.partition.mode` | If the ingestion pipeline creates partitions dynamically, these settings must be enabled. |

If the script is templated (e.g., via Airflow or a CI pipeline), the HDFS URI may be parameterised; check the surrounding workflow for variable substitution.

---

## 8. Suggested Improvements (TODO)

1. **Replace generic `string` types with precise data types** – e.g., `eventtimestamp` → `timestamp`, `dpitransactionid` → `bigint`. This reduces casting in downstream queries and improves query planning.

2. **Add column comments and a table comment** to document the meaning of each field and the purpose of the table, aiding future developers and data catalog tools.

*(Both changes can be applied via an `ALTER TABLE ... REPLACE COLUMNS` statement after the initial creation, followed by a data back‑fill if needed.)*