**File:** `mediation-ddls\msisdn_level_daily_usage_aggr.hql`  

---

## 1. High‑level Summary
This script defines an **external Hive table** `mnaas.msisdn_level_daily_usage_aggr`. The table stores daily, per‑MSISDN usage aggregates (data, voice, SMS, cost, device and commercial‑offer metadata) in **Parquet** format on HDFS. It is **partitioned by `partition_date`** (string) to enable efficient daily loads and queries. The table is intended to be populated by upstream ETL jobs that calculate per‑MSISDN usage and to be consumed by downstream reporting, billing, or analytics pipelines.

---

## 2. Core Artefacts

| Artefact | Type | Responsibility |
|----------|------|-----------------|
| `CREATE EXTERNAL TABLE mnaas.msisdn_level_daily_usage_aggr` | Hive DDL | Registers the schema, storage format, location and partitioning for the daily usage aggregate data. |
| Columns (≈50) | Data fields | Capture customer, network, device, commercial‑offer, usage (data/voice/SMS), cost, and audit information for each MSISDN. |
| `PARTITIONED BY (partition_date STRING)` | Partition spec | Enables daily ingestion and pruning of data. |
| `ROW FORMAT SERDE … ParquetHiveSerDe` / `STORED AS INPUTFORMAT … Parquet` | Storage definition | Persists data as columnar Parquet files for compression and query performance. |
| `LOCATION 'hdfs://NN-HA1/.../msisdn_level_daily_usage_aggr'` | Physical location | Points to the HDFS directory where the external table files reside. |
| `TBLPROPERTIES` | Table metadata | Controls stats collection, purge behaviour, Impala catalog integration, and Spark schema hints. |

*No procedural code (functions, procedures, scripts) is present in this file.*

---

## 3. Inputs, Outputs & Side‑effects

| Aspect | Details |
|--------|---------|
| **Inputs** | - Parquet files written to the HDFS location by upstream batch jobs (e.g., Spark, Hive INSERT/INSERT OVERWRITE, or custom ETL). <br> - Partition values (`partition_date`) supplied by those jobs. |
| **Outputs** | - Hive metastore entry for `msisdn_level_daily_usage_aggr`. <br> - Logical view of the data for downstream queries (Hive, Impala, Spark). |
| **Side‑effects** | - Registers an **external** table; dropping the table will **purge** the underlying HDFS files because of `'external.table.purge'='true'`. <br> - Updates Hive metastore statistics (if later collected). |
| **Assumptions** | - HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/msisdn_level_daily_usage_aggr` exists and is writable by the Hive/Impala service accounts. <br> - Hive/Impala versions support the Parquet SerDe and the listed table properties. <br> - Partition management (adding new `partition_date` values) is performed by downstream load scripts. |

---

## 4. Integration Points

| Connected Component | Interaction |
|---------------------|-------------|
| **Upstream aggregation jobs** (e.g., `msisdn_daily_usage_agg_spark.py`, `move_msisdn_usage_daily.hql`) | Write Parquet files into the table’s HDFS location and add the corresponding partition (`ALTER TABLE … ADD PARTITION (partition_date='2024‑12‑03')`). |
| **Downstream reporting / billing** (e.g., `billing_daily_usage_view.hql`, `usage_cost_calc.py`) | Query the table directly (`SELECT … FROM mnaas.msisdn_level_daily_usage_aggr WHERE partition_date = …`). |
| **Metadata / catalog services** (Impala catalog, Hive Metastore) | Consume the `TBLPROPERTIES` for stats and versioning; Impala events reference the `catalogServiceId` and `catalogVersion`. |
| **Orchestration layer** (Airflow, Oozie) | Executes this DDL as part of a “create tables” DAG/task before any load jobs run. |
| **Security / ACLs** | Relies on HDFS ACLs and Hive/Impala role permissions defined elsewhere (e.g., `hive-site.xml`, Ranger policies). |

*Because the script only creates the table, any script that **adds partitions** or **loads data** must reference the same database (`mnaas`) and location.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream jobs add/remove columns without updating this DDL. | Query failures, data loss, or corrupted Parquet files. | Enforce schema version control; run a nightly schema‑validation job (`spark-submit --dry-run` or Hive `DESCRIBE EXTENDED`). |
| **Partition explosion** – missing `ADD PARTITION` leads to data being written to the default partition, causing full‑table scans. | Performance degradation, storage bloat. | Automate partition addition in the load job; verify partition existence after each load (`SHOW PARTITIONS`). |
| **External table purge** – accidental `DROP TABLE` removes raw Parquet files. | Irrecoverable data loss. | Restrict DROP privileges; enable Hive metastore backup; add a pre‑drop safeguard script. |
| **Insufficient HDFS space** – daily aggregates grow quickly. | Job failures, back‑pressure on upstream pipelines. | Monitor HDFS usage; implement TTL/archival policy (e.g., move partitions older than 90 days to cold storage). |
| **Permission mismatches** – Hive/Impala service accounts lack write access to the HDFS path. | Table creation or data load failures. | Validate ACLs during deployment; include a health‑check step (`hdfs dfs -test -w <path>`). |

---

## 6. Running / Debugging the Script

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/msisdn_level_daily_usage_aggr.hql
   # or via Beeline
   beeline -u jdbc:hive2://<hs2-host>:10000 -f msisdn_level_daily_usage_aggr.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.msisdn_level_daily_usage_aggr;
   DESCRIBE FORMATTED mnaas.msisdn_level_daily_usage_aggr;
   ```

3. **Check HDFS location**  
   ```bash
   hdfs dfs -ls hdfs://NN-HA1/user/hive/warehouse/mnaas.db/msisdn_level_daily_usage_aggr
   ```

4. **Add a test partition** (simulating an upstream load)  
   ```sql
   ALTER TABLE mnaas.msisdn_level_daily_usage_aggr
     ADD IF NOT EXISTS PARTITION (partition_date='2024-12-03')
     LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/msisdn_level_daily_usage_aggr/partition_date=2024-12-03';
   ```

5. **Debug common issues**  
   * *“Table already exists”* – use `CREATE EXTERNAL TABLE IF NOT EXISTS`.  
   * *“Permission denied”* – verify HDFS ACLs and Hive service principal.  
   * *“Invalid SerDe”* – ensure Hive/Impala versions include the Parquet SerDe libraries.

---

## 7. External Configuration / Environment Dependencies

| Item | How it is used |
|------|----------------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` URI; must resolve in the cluster network. |
| Hive Metastore connection (usually via `hive-site.xml`) | Required for the DDL to register the table. |
| Impala catalog service ID (`b9c115786e934562:94a82089945b48d6`) | Stored as a table property for Impala synchronization; not directly referenced in the script. |
| Spark/Impala schema hints (`spark.sql.sources.*`) | Enable Spark to infer the schema without scanning the data; useful for downstream Spark jobs. |
| Potential environment variables (e.g., `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) | Needed by the Hive/Beeline client that runs the script. |

*If the deployment environment changes (e.g., a new namenode address), the `LOCATION` line must be updated accordingly.*

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard**  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.msisdn_level_daily_usage_aggr ( ... )
   ```
   Prevents failures on re‑run during CI/CD or disaster‑recovery restores.

2. **Separate partition‑management script**  
   Create a lightweight Hive/Beeline script (e.g., `add_partition_msisdn_usage.hql`) that receives a `partition_date` argument and adds the partition with the correct HDFS sub‑directory. This isolates partition logic from the DDL and reduces the risk of missing partitions.

---