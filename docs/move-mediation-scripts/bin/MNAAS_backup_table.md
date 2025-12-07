**File:** `move-mediation-scripts/bin/MNAAS_backup_table.py`

---

## 1. Summary
`MNAAS_backup_table.py` is a PySpark utility that creates a backup copy of a production Hive/Databricks table (`mnaas.<table_name>`) by reading raw CSV files from a supplied folder, applying the source table’s schema, adding year/month partition columns derived from the `partition_date` field, and overwriting (or creating) a backup table named `mnaas.<table_name>_bkp`. The backup table is partitioned by `year_part` and `month_part` to support downstream queries and retention policies.

---

## 2. Key Functions & Responsibilities

| Name | Type | Responsibility |
|------|------|-----------------|
| `load_backup(table_name, input_path_folder)` | Function | * Retrieves the schema of the live table (`mnaas.<table_name>`).<br>* Reads CSV files from `input_path_folder` using that schema (delimiter `;`, no header).<br>* Adds `year_part` and `month_part` columns derived from `partition_date`.<br>* Drops any existing backup table (`mnaas.<table_name>_bkp`).<br>* Writes the transformed DataFrame to the backup table, partitioned by `year_part`/`month_part` and using `overwrite` mode. |
| `__main__` block | Script entry point | * Parses two positional arguments: `<table_name>` and `<input_path_folder>`.<br>* Instantiates a SparkSession named “Backup Table”.<br>* Calls `load_backup`. |

*No classes are defined; the script is functional‑style.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | 1. **Command‑line argument 1** – `table_name` (string, e.g., `TrafficDetails`).<br>2. **Command‑line argument 2** – `input_path_folder` (HDFS/S3/local path containing CSV files). |
| **External Services / Resources** | • Hive/Metastore catalog `mnaas` (source and backup tables).<br>• Underlying storage for CSV files (HDFS, S3, ADLS, etc.).<br>• Spark cluster (YARN, Kubernetes, EMR, etc.). |
| **Outputs** | • Hive/Databricks table `mnaas.<table_name>_bkp` populated with the CSV data, partitioned by `year_part`/`month_part`.<br>• Spark job logs (stdout/stderr). |
| **Side Effects** | • Drops the existing backup table (if any).<br>• Overwrites data in the backup table – previous backup is lost. |
| **Assumptions** | • Source table `mnaas.<table_name>` exists and contains at least one row (used to fetch schema).<br>• CSV files match the source schema exactly (order, data types).<br>• Column `partition_date` exists and is a valid timestamp/date in every row.<br>• The Spark session has permission to read/write the Hive catalog and the file system path. |

---

## 4. Integration Points & Connectivity

| Connected Component | How the Connection Is Made |
|---------------------|-----------------------------|
| **Data ingestion scripts** (e.g., `MNAAS_Traffic_tbl_with_nodups_loading.sh`) | Those scripts generate the CSV files that this backup script consumes. The folder path is passed as the second argument. |
| **Orchestration layer** (Airflow, Oozie, cron) | Typically invoked as a Spark submit task after a successful load or before a destructive operation. Example command: `spark-submit MNAAS_backup_table.py TrafficDetails /data/mnaas/traffic/details`. |
| **Retention / cleanup jobs** (e.g., `MNAAS_backup_file_retention.sh`) | May rely on the partitioned backup tables for TTL policies; they reference the same `year_part`/`month_part` columns. |
| **Monitoring / alerting** (e.g., `MNAAS_Traffic_Loading_Progress_Trend_Mail.sh`) | Could query the backup table to verify that a backup exists before proceeding with downstream processing. |
| **Configuration files** | No explicit config file is read, but Spark defaults (e.g., `spark.sql.warehouse.dir`, Hive metastore URI) are expected to be set via environment variables or Spark submit options. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – CSV columns no longer match source schema. | Job fails or writes corrupted data. | Add schema validation step (compare column names/types) and fail fast with a clear error message. |
| **Missing `partition_date`** – Null or malformed dates. | Rows filtered out (`partition_date is not null`) leading to data loss. | Validate that the column exists and contains non‑null values before filtering; optionally route bad rows to a “reject” folder. |
| **Overwrite of existing backup** – Accidental data loss. | Previous backup is irretrievably lost. | Change `mode("overwrite")` to `mode("append")` after a `DROP TABLE IF EXISTS` guard, or implement a versioned backup naming scheme (e.g., `_bkp_YYYYMMDD`). |
| **Large volume causing OOM** – Full dataset read into a single executor. | Spark job crashes. | Tune Spark executor memory, enable `spark.sql.shuffle.partitions`, or process data in chunks (e.g., incremental loads). |
| **Insufficient permissions** – Cannot read CSV or write Hive table. | Job aborts. | Verify service account rights; include a pre‑flight permission check. |
| **No logging / error handling** – Operators cannot diagnose failures. | Silent failures or ambiguous logs. | Wrap main logic in try/except, log to stdout and optionally to a central logging system (e.g., CloudWatch, ELK). |

---

## 6. Execution & Debugging Guide

### 6.1 Typical Run (Production)

```bash
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --conf spark.sql.warehouse.dir=/user/hive/warehouse \
  move-mediation-scripts/bin/MNAAS_backup_table.py \
  TrafficDetails \
  hdfs:///data/mnaas/traffic/details/
```

*Replace `--master` and other Spark configs with the environment‑specific values.*

### 6.2 Development / Local Debug

```bash
export PYSPARK_PYTHON=python3
spark-submit \
  --master local[*] \
  move-mediation-scripts/bin/MNAAS_backup_table.py \
  TrafficDetails \
  /tmp/local_test/traffic_details/
```

*Use a small subset of CSV files to verify schema mapping and partition column creation.*

### 6.3 Debug Steps

1. **Validate Input Path** – `hdfs dfs -ls <path>` (or `aws s3 ls` etc.) to ensure files exist.
2. **Check Source Table Schema** – `spark.sql("DESCRIBE mnaas.<table_name>").show()`.
3. **Run with `--conf spark.sql.shuffle.partitions=10`** to reduce shuffle overhead for small test data.
4. **Inspect Spark UI** (usually at `http://<driver-node>:4040`) for stage details and any skew.
5. **Verify Backup Table** – after run, execute `spark.sql("SELECT count(*) FROM mnaas.<table_name>_bkp").show()` and compare with source row count.

---

## 7. External Configuration & Environment Variables

| Variable / Config | Usage |
|-------------------|-------|
| `SPARK_CONF_DIR` / `spark-submit` options | Determines Hive metastore URI, warehouse location, authentication (Kerberos, IAM). |
| `PYSPARK_PYTHON` | Python interpreter for executors (must match the version used to develop the script). |
| `HADOOP_CONF_DIR` | Provides HDFS/S3 credentials and connection details. |
| No explicit config file is read by the script itself; all parameters are passed via command‑line arguments or Spark’s runtime configuration. |

---

## 8. Suggested Improvements (TODO)

1. **Add Robust Validation & Logging**  
   - Implement a schema‑comparison routine that aborts if the CSV column order/types diverge from the source table.  
   - Use Python’s `logging` module to emit structured logs (INFO, WARN, ERROR) and write them to a central log sink.

2. **Make Backup Table Naming Configurable & Versioned**  
   - Introduce an optional third argument `backup_suffix` (default `_bkp`).  
   - Append a timestamp (e.g., `_bkp_20251204`) to avoid accidental overwrites and to support point‑in‑time restores.

---