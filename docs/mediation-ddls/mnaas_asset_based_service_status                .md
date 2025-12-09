# Summary
Creates the Hive managed table **`mnaas.asset_based_service_status`** used to store raw asset‑based service events. The table is partitioned by `event_month` and configured for insert‑only transactional writes. Data resides in HDFS under the `mnaas.db` warehouse path.

# Key Components
- **Table definition** – `asset_based_service_status` with 15 string columns (`sourceid` … `record_insert_time`).
- **Partition column** – `event_month` (string) for monthly data isolation.
- **SerDe** – `LazySimpleSerDe` for delimited text input.
- **File format** – `TextInputFormat` / `HiveIgnoreKeyTextOutputFormat`.
- **Table properties** –  
  - `bucketing_version='2'`  
  - `transactional='true'` (insert‑only)  
  - `transient_lastDdlTime='1736953657'`

# Data Flow
| Stage | Description |
|-------|-------------|
| **Source** | External event producers (e.g., network elements, billing adapters) write delimited text files to a landing zone. |
| **Ingestion** | ETL job (Spark/MapReduce/Hive INSERT) reads landing files, transforms as needed, and inserts into `asset_based_service_status` using `INSERT INTO … PARTITION (event_month=…)`. |
| **Storage** | Rows are stored as plain text files under `hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/asset_based_service_status/event_month=YYYYMM/`. |
| **Consumption** | Downstream analytics, reporting, or SLA monitoring jobs query the table, optionally filtering on `event_month`. |
| **Side Effects** | Transaction log updates in Hive metastore (ACID insert‑only). No external queues are touched directly by this DDL. |

# Integrations
- **Hive Metastore** – registers the table metadata; accessed by all Hive/Presto/Trino queries.
- **ETL orchestration** – Airflow, Oozie, or Azure Data Factory jobs that stage files and execute `INSERT` statements.
- **Monitoring** – Metrics collected from HiveServer2 and HDFS usage for the table path.
- **Downstream consumers** – BI dashboards, anomaly detection pipelines, and billing reconciliation services that read the table.

# Operational Risks
- **String‑only schema** – lack of type enforcement may cause downstream parsing errors. *Mitigation*: enforce validation in ingestion jobs.
- **Partition drift** – missing or mis‑named `event_month` partitions lead to data landing in the default partition. *Mitigation*: automate partition creation and enforce naming conventions.
- **Insert‑only ACID** – updates/deletes are not supported; stale records accumulate. *Mitigation*: schedule periodic compaction or archival jobs.
- **Uncompressed storage** – high disk I/O and storage cost. *Mitigation*: enable compression (e.g., ORC/Parquet) in a future migration.

# Usage
```bash
# Create the table (idempotent)
hive -f mediation-ddls/mnaas_asset_based_service_status.hql

# Insert data (example)
hive -e "
INSERT INTO mnaas.asset_based_service_status PARTITION(event_month='202410')
SELECT
  sourceid,
  eventid,
  eventtimestamp,
  entityid_iccid,
  eventclass,
  dpibillingid,
  dpitransactionid,
  tenancyid_secsid,
  imsi,
  start_time,
  dpitransactionstatus,
  created_on,
  last_updated_on,
  apnname,
  record_insert_time
FROM staging_raw_events
WHERE event_month='202410';
"
```
To debug, enable Hive logging (`set hive.exec.verbose=true;`) and inspect the HDFS partition directory.

# Configuration
- **Hive environment variables** – `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`.
- **Metastore connection** – `hive.metastore.uris` (typically `thrift://<host>:9083`).
- **HDFS namenode** – `fs.defaultFS=hdfs://NN-HA1`.
- **Warehouse directory** – defined by `hive.metastore.warehouse.dir` (defaults to `/warehouse/tablespace/managed/hive`).

# Improvements
1. **Data typing** – Replace generic `string` columns with appropriate Hive types (e.g., `bigint` for timestamps, `int` for IDs) to enable predicate push‑down and reduce parsing errors.
2. **Storage format** – Migrate to a columnar, compressed format such as ORC with `STORED AS ORC` and enable `orc.compress=ZSTD` to improve query performance and reduce storage footprint.