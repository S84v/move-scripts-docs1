**File:** `move-mediation-scripts/bin/MNAAS_msisdn_jlr_rate_per_country.py`

---

## 1. High‑Level Summary
This Python‑Spark job reads a list of partition dates from a configuration file, builds a dynamic Hive SQL query to pull MSISDN‑level daily usage records for a specific TCL SECS ID (`24048`) and device types (`PIVI`, `VDC`). It joins the usage data with the `mnaas.jlr_rate_per_country` reference table to enrich each record with the applicable rate, then writes the enriched rows back to the Hive table `mnaas.msisdn_jlr_rate_per_country`, partitioned by `partition_date`. The script is part of the “MSISDN JLR rate per country” aggregation pipeline that runs after the daily usage aggregation step.

---

## 2. Important Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `sql_query()` | • Reads the file `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/MNAAS_MSISDN_Aggr_Partitions_testing` (or the production counterpart). <br>• Extracts a list of partition dates. <br>• Constructs a Hive SQL string that selects distinct usage rows for the target `tcl_secs_id` and device types, limited to the supplied dates. |
| `msisdn_jlr_rate(sql_query)` | • Starts a Spark session named **msisdn_jlr_rate**. <br>• Sets Hive‑related Spark configs (`dynamic.partition`, `dynamic.partition.mode`). <br>• Executes the generated SQL to obtain a DataFrame (`df`). <br>• Registers `df` as a temporary view `msisdn_pivi_vdc_data`. <br>• Performs an inner join with `mnaas.jlr_rate_per_country` on `tcl_secs_id`, `ratinggroupid`, and case‑insensitive `country`. <br>• Writes the resulting DataFrame (`df1`) in **append** mode, partitioned by `partition_date`, to Hive table `mnaas.msisdn_jlr_rate_per_country`. <br>• Stops the Spark session. |

No classes are defined; the script is functional‑style.

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Input Files** | `MNAAS_MSISDN_Aggr_Partitions_testing` (or the production file `MNAAS_MSISDN_Aggr_Partitions_Daily_Uniq`). Contains one partition date per line, e.g. `2024-11-30`. |
| **Hive Tables Read** | `mnaas.msisdn_level_daily_usage_aggr` (source usage data). <br>`mnaas.jlr_rate_per_country` (rate reference). |
| **Hive Table Written** | `mnaas.msisdn_jlr_rate_per_country` – enriched data, **append** mode, partitioned by `partition_date`. |
| **External Services** | Hadoop/Hive metastore, YARN (or Spark standalone) cluster, HDFS for the configuration file. |
| **Assumptions** | • The configuration file exists and is readable by the user running the script. <br>• The Hive tables have the columns referenced in the SQL (e.g., `call_month_name`, `country`, `vin`, `device_type`, `sim`, `data_usage_in_mb`, `ratinggroupid`, `tcl_secs_id`, `partition_date`). <br>• The Spark environment is pre‑configured with Hive support (`spark.sql.catalogImplementation=hive`). <br>• No duplicate partitions are written because `append` mode is used; downstream processes must handle possible duplicates. |
| **Side Effects** | • Creates/updates Hive table `mnaas.msisdn_jlr_rate_per_country`. <br>• Emits Spark logs to stdout/stderr and possibly to YARN/cluster log aggregation. |
| **Return Values** | None – script exits with status 0 on success, non‑zero on uncaught exception. |

---

## 4. Integration with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **Daily usage aggregation scripts** (`MNAAS_msisdn_daily_aggr_*.sh` or similar) | Produce the source Hive table `mnaas.msisdn_level_daily_usage_aggr`. This script consumes that table. |
| **Rate reference loader** (`MNAAS_load_jlr_rate_per_country.sh` or equivalent) | Populates `mnaas.jlr_rate_per_country`. Must run before this job. |
| **Partition list generator** (`MNAAS_generate_partition_file.py` or a Bash wrapper) | Writes the dates file read by `sql_query()`. Usually executed by a scheduler (e.g., Oozie, Airflow) prior to this job. |
| **Downstream consumption** (`MNAAS_msisdn_jlr_rate_report.py`, BI extracts) | Reads `mnaas.msisdn_jlr_rate_per_country` for reporting or further transformation. |
| **Scheduler** (e.g., Oozie, Airflow, cron) | Invokes this script as a Spark submit command, e.g., `spark-submit MNAAS_msisdn_jlr_rate_per_country.py`. The scheduler also ensures the configuration file is refreshed daily. |

*Note:* The script lives in the same `move-mediation-scripts/bin` directory as other “MNAAS” jobs, indicating a shared deployment and versioning strategy.

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed partition file** | Job aborts, downstream tables not refreshed. | Add validation: exit with clear error if file empty or contains invalid dates. |
| **Hive table schema drift** (column rename, type change) | Spark SQL failures, data loss. | Implement schema validation before query execution; version control Hive DDL. |
| **Duplicate rows due to `append` mode** | Over‑counting in downstream reports. | Use `INSERT OVERWRITE` for full rebuild or add deduplication logic (e.g., `INSERT INTO … SELECT DISTINCT …`). |
| **Large date list causing excessively long IN clause** | Query planner may degrade performance. | Switch to a temporary Hive/Parquet table containing the dates and join instead of `IN (…)`. |
| **Spark session leaks** (if `spark.stop()` not reached) | Resource exhaustion on the cluster. | Ensure `finally` block always executes; consider using context manager (`with SparkSession.builder... as spark:`). |
| **Hard‑coded TCL SECS ID (`24048`)** | Inflexibility across environments. | Externalize the ID to a config file or environment variable. |

---

## 6. Example Run / Debug Procedure

### 6.1 Typical Production Invocation
```bash
# Assuming Spark is on the PATH and the user has HDFS/Hive permissions
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --conf spark.yarn.queue=default \
  /opt/mnaas/move-mediation-scripts/bin/MNAAS_msisdn_jlr_rate_per_country.py
```

*The scheduler (Airflow/Oozie) usually wraps the above command.*

### 6.2 Local Debug / Unit Test
1. **Set up a local Spark session** (e.g., `spark-shell` or `pyspark`) with Hive support enabled.
2. **Create a mock partition file**:
   ```bash
   echo "2024-11-30" > /tmp/partitions.txt
   export MNAAS_PARTITION_FILE=/tmp/partitions.txt   # modify script to read env var if needed
   ```
3. **Run the script directly**:
   ```bash
   python3 MNAAS_msisdn_jlr_rate_per_country.py
   ```
4. **Inspect logs** – Spark UI (usually at `http://<driver-node>:4040`) shows the generated query and job stages.
5. **Validate output**:
   ```sql
   SELECT COUNT(*) FROM mnaas.msisdn_jlr_rate_per_country
   WHERE partition_date = '2024-11-30';
   ```

### 6.3 Debugging Tips
- Insert `print(df.schema)` after each `spark.sql()` to verify column names/types.
- Use `df.explain(True)` to view the physical plan for the join.
- Capture the generated SQL (already printed) and run it manually in Hive Beeline to isolate issues.

---

## 7. External Configuration, Environment Variables & Files

| Item | Purpose | Current Usage |
|------|---------|---------------|
| `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/MNAAS_MSISDN_Aggr_Partitions_testing` (or production counterpart) | Supplies the list of `partition_date` values to process. | Hard‑coded path inside `sql_query()`. |
| Spark/Hive configs (`spark.sql.debug.maxToStringFields`, `hive.exec.dynamic.partition*`) | Enables dynamic partitioning and generous debug output. | Set programmatically in `msisdn_jlr_rate()`. |
| Potential env vars (not currently used) | Could externalize `tcl_secs_id`, device types, file paths, or Hive DB name. | Not present – an improvement opportunity. |

---

## 8. Suggested TODO / Improvements

1. **Externalize hard‑coded parameters**  
   - Move `tcl_secs_id`, device type list, and the partition‑file path to a properties file or environment variables. This will allow the same script to be reused across environments (dev/test/prod) without code changes.

2. **Replace the `IN (…)` date filter with a join**  
   - Create a temporary Hive/Parquet table (or Spark DataFrame) from the dates file and join on `partition_date`. This scales better for large date sets and avoids SQL length limits.

---