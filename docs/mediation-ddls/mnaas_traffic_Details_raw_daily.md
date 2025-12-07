**File:** `mediation-ddls\mnaas_traffic_Details_raw_daily.hql`  

---

## 1. High‑Level Summary
This HQL script creates an **external Hive/Impala table** named `mnaas.traffic_details_raw_daily`. The table stores raw Call Detail Records (CDRs) received from MVNO partners in **Parquet** format, partitioned by a string `partition_date`. It is the canonical source for daily raw traffic data that downstream aggregation, billing, and reporting jobs (e.g., `mnaas_traffic_Details_summary`, `mnaas_ra_*` reports) query.

---

## 2. Core Object(s) Defined
| Object | Type | Responsibility |
|--------|------|-----------------|
| `traffic_details_raw_daily` | External Hive table | Persists raw CDR rows exactly as received from upstream ingestion pipelines; provides a stable schema for analytics jobs. |
| `createtab_stmt` (implicit) | DDL statement | Encapsulates the full `CREATE EXTERNAL TABLE …` definition, including column list, partitioning, SerDe, storage format, location, and table properties. |

*No procedural code (functions, classes) is present; the file is purely declarative.*

---

## 3. Inputs, Outputs & Side Effects  

| Aspect | Details |
|--------|---------|
| **Input data** | Parquet files written to HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_raw_daily` by upstream ingestion jobs (e.g., SFTP pullers, Kafka‑to‑HDFS connectors). Files are expected to follow the column order defined in the DDL. |
| **Output** | A Hive/Impala external table exposing the raw CDR rows to any downstream Spark, Hive, or Impala query. No data is copied; the table is a pointer to the files on HDFS. |
| **Side effects** | - Registers metadata in the Hive metastore. <br>- Because `external.table.purge='true'`, dropping the table will also delete the underlying HDFS files. <br>- Table properties disable automatic stats updates (`DO_NOT_UPDATE_STATS='true'`). |
| **Assumptions** | - The HDFS location exists and is writable by the Hive service account. <br>- Files are written in **Parquet** using the same schema (no extra columns, compatible types). <br>- Partition column `partition_date` is populated (e.g., `2024-12-04`). <br>- Hive metastore is reachable from the execution environment. |

---

## 4. Integration Points (How This Table Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **Ingestion pipelines** (e.g., `mnaas_ingest_cdrs.sh`, Spark jobs) | Write daily Parquet files to the table’s HDFS location; may also create the partition directory (`partition_date=YYYY-MM-DD`). |
| **Aggregation scripts** (e.g., `mnaas_traffic_Details_summary.hql`, `mnaas_ra_*` reports) | SELECT from `traffic_details_raw_daily` (often filtered by `partition_date`) to compute usage, billing, and KPI aggregates. |
| **Data quality / rejection handling** | The column `rejectcode` is used by downstream validation jobs to isolate malformed records. |
| **Metadata / catalog services** (Impala catalog, Hive metastore) | The table definition is stored here; any schema change requires a DDL update and a catalog refresh. |
| **Retention / purge jobs** | Periodic scripts may drop old partitions (e.g., `ALTER TABLE … DROP IF EXISTS PARTITION (partition_date='2023-01-01')`). Because the table is external, the underlying files are removed automatically. |
| **Security / access control** | Permissions on the HDFS directory and Hive metastore ACLs control who can read/write the raw data. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream source adds/removes columns. | Queries downstream may fail or return wrong results. | Enforce schema versioning: upstream pipelines must validate against the current DDL; add a CI check that compares incoming Parquet schema to the Hive table schema. |
| **Partition explosion** – one partition per day without cleanup. | Storage bloat, degraded query performance. | Implement a retention policy (e.g., keep 90 days) that drops old partitions automatically. |
| **Incorrect file format** – non‑Parquet files land in the directory. | Table becomes unreadable; job failures. | Add a pre‑load validation step that checks file extensions and Parquet footers before moving files to the location. |
| **External table purge** – accidental `DROP TABLE` removes raw files. | Permanent data loss. | Restrict DROP privileges; use `DROP IF EXISTS` only in controlled maintenance windows; enable backup snapshots of the HDFS path. |
| **Missing partition metadata** – files placed without creating the Hive partition. | Queries return empty result sets. | Ingestion jobs must issue `ALTER TABLE … ADD PARTITION (partition_date='...') LOCATION '…'` or rely on Hive’s `MSCK REPAIR TABLE`. Automate this step. |
| **Performance degradation** – large single files or skewed data distribution. | Slow scans, high memory pressure. | Encourage upstream to write files ≤ 256 MB and use balanced partitioning (e.g., by date + MVNO). |

---

## 6. Running / Debugging the Script  

| Step | Command | Purpose |
|------|---------|---------|
| **Create/Refresh table** | `hive -f mediation-ddls/mnaas_traffic_Details_raw_daily.hql` <br>or <br>`impala-shell -i <impala-host> -f mediation-ddls/mnaas_traffic_Details_raw_daily.hql` | Executes the DDL, registers the external table. |
| **Validate existence** | `SHOW CREATE TABLE mnaas.traffic_details_raw_daily;` | Confirms the table definition matches expectations. |
| **Inspect partitions** | `SHOW PARTITIONS mnaas.traffic_details_raw_daily;` | Lists available `partition_date` values. |
| **Sample query** | `SELECT COUNT(*) FROM mnaas.traffic_details_raw_daily WHERE partition_date='2024-12-04';` | Verifies data is present for a given day. |
| **Check underlying files** | `hdfs dfs -ls /user/hive/warehouse/mnaas.db/traffic_details_raw_daily/partition_date=2024-12-04/` | Ensures Parquet files exist and are readable. |
| **Refresh metadata** (if files added manually) | `MSCK REPAIR TABLE mnaas.traffic_details_raw_daily;` | Adds missing partitions to the metastore. |
| **Debug schema mismatch** | `hdfs dfs -cat <file> | parquet-tools schema -` (or Spark `spark.read.parquet(...).printSchema()`) | Compare actual Parquet schema to the Hive DDL. |

---

## 7. External Configuration / Environment Variables  

| Config | Where Used | Description |
|--------|------------|-------------|
| `NN-HA1` (NameNode HA address) | HDFS `LOCATION` clause | Points to the HDFS cluster where raw files reside. Typically supplied via a cluster‑wide Hadoop configuration (`fs.defaultFS`). |
| Hive Metastore connection (`hive.metastore.uris`) | Hive/Impala services | Determines where the table metadata is stored. |
| Impala catalog service ID (`impala.events.catalogServiceId`) | Table property (auto‑generated) | Used internally by Impala for catalog synchronization; not manually set. |
| `spark.sql.create.version` / `spark.sql.sources.*` properties | Table properties | Reflect the Spark version that originally created the table; useful for compatibility checks. |
| No explicit env‑vars are referenced in the script; all values are hard‑coded. If the deployment environment changes (e.g., different HDFS namespace), the script must be edited accordingly. |

---

## 8. Suggested Improvements (TODO)

1. **Add a partition‑creation helper** – a small Bash/Python wrapper that:
   * Receives a processing date,
   * Creates the HDFS directory,
   * Writes an empty `_SUCCESS` marker,
   * Executes `ALTER TABLE … ADD PARTITION … LOCATION …`,
   * Logs the operation.  
   This will eliminate reliance on `MSCK REPAIR TABLE` and guarantee partition metadata is always present.

2. **Introduce schema versioning column** – add a nullable `schema_version` string (or integer) to the table. Downstream jobs can then detect rows written with an older schema and apply appropriate transformation logic, reducing breakage when upstream adds new fields.  

--- 

*End of documentation.*