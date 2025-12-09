# Summary
`MOVE_CURR_MONTH_COUNT` is an auto‑generated Sqoop ORM class that maps the Oracle table **MOVE_CURR_MONTH_COUNT** to a Hadoop‑compatible record. It enables bulk import of monthly SIM‑count metrics (business unit, product status, SIM count, status date) into Hive/Impala tables used by the Move‑Mediation nightly pipelines.

# Key Components
- **Class `MOVE_CURR_MONTH_COUNT`** – extends `SqoopRecord`, implements `DBWritable` & `Writable`.
- **Field definitions** – `BUSS_UNIT_ID`, `PROD_STATUS`, `COUNT_SIM`, `STATUS_DATE`.
- **Setters map (`setters`)** – runtime reflection for dynamic field assignment.
- **`readFields(ResultSet)` / `write(PreparedStatement)`** – JDBC ↔ object serialization.
- **`readFields(DataInput)` / `write(DataOutput)`** – Hadoop Writable serialization.
- **`toString(DelimiterSet)`** – CSV/TSV rendering for debugging or downstream text output.
- **`parse(...)`** – parses delimited text back into object fields.
- **`getFieldMap()` / `setField(String,Object)`** – generic field access for Sqoop runtime.

# Data Flow
| Stage | Input | Processing | Output |
|-------|-------|-------------|--------|
| **Sqoop import** | Oracle `MOVE_CURR_MONTH_COUNT` rows (via JDBC) | `readFields(ResultSet)` populates object fields | Hadoop Writable records streamed to HDFS |
| **HDFS staging** | Writable records written by Sqoop to `/user/sqoop/...` | Serialized via `write(DataOutput)` | Partitioned files (default CSV) |
| **Hive load** | Staged files | Hive external table / `LOAD DATA` command (invoked by downstream scripts) | Hive table `mnaas.move_curr_month_count` (or similar) partitioned by `STATUS_DATE` |

Side effects: network I/O to Oracle, HDFS write, possible temporary local disk usage for map tasks.

# Integrations
- **Sqoop job definitions** – referenced in Move‑Mediation ETL orchestration (e.g., Oozie or Airflow) that schedule the import.
- **Hive scripts** – downstream Hive/Beeline jobs (e.g., `med_traffic_file_recon.hql`) consume the imported table for reconciliation metrics.
- **Spark execution engine** – Hive tables are later read by Spark jobs for analytics.
- **Configuration** – Hive session settings (dynamic partitioning) and Oracle connection parameters are supplied via Sqoop job properties.

# Operational Risks
- **Schema drift** – changes in Oracle column types break serialization (`COUNT_SIM` precision). *Mitigation*: version‑controlled Sqoop job, regenerate class on schema change.
- **Data type overflow** – `BigDecimal` precision mismatch with Hive `DECIMAL`. *Mitigation*: enforce matching precision/scale in Hive DDL.
- **Network latency / timeouts** – large imports may exceed JDBC timeout. *Mitigation*: tune `--fetch-size`, increase `sqoop.import.timeout`.
- **Partial imports** – job failure leaves incomplete HDFS files. *Mitigation*: use Sqoop `--delete-target-dir` and idempotent Hive `INSERT OVERWRITE`.

# Usage
```bash
# 1. Compile (if modifying)
javac -cp $(sqoop classpath) MOVE_CURR_MONTH_COUNT.java

# 2. Run Sqoop import (example)
sqoop import \
  --connect jdbc:oracle:thin:@//oracle-host:1521/PROD \
  --username MOVETEL \
  --password-file hdfs:///user/keys/oracle.pass \
  --table MOVE_CURR_MONTH_COUNT \
  --target-dir /user/hive/warehouse/mnaas.db/move_curr_month_count \
  --as-textfile \
  --fields-terminated-by ',' \
  --null-string '\\N' \
  --null-non-string '\\N' \
  --class-name MOVE_CURR_MONTH_COUNT \
  --map-column-java COUNT_SIM=java.math.BigDecimal \
  --split-by BUSS_UNIT_ID \
  --num-mappers 4
```
*Debug*: instantiate class in a REPL, call `parse("U001,ACTIVE,12345,2025-01-31")` and inspect fields.

# Configuration
- **Environment variables**: `SQOOP_CONF_DIR`, `HADOOP_CONF_DIR`.
- **Sqoop properties**: `sqoop.import.timeout`, `sqoop.fetch.size`.
- **Hive settings** (used downstream): `hive.exec.dynamic.partition=true`, `hive.exec.max.dynamic.partitions=2000`.
- **Oracle credentials**: stored in HDFS credential file referenced by `--password-file`.

# Improvements
1. **Schema versioning** – generate a versioned package (e.g., `com.mnaas.sqoop.v1`) and embed a schema hash to detect drift at runtime.
2. **Custom Writable** – replace generic CSV serialization with Avro/Parquet to reduce storage size and improve downstream Spark performance.