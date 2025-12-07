**High‑Level Summary**  
`traffic_nodups_summation.py` is a Spark‑based data‑processing job that reads raw, de‑duplicated CDR records from the Hive table `mnaas.traffic_details_raw_daily_with_no_dups` for a supplied date range, aggregates usage per SIM (separately for Data, Voice and SMS), normalises a handful of string fields, enriches the result with customer names from `mnaas.org_details`, and writes the consolidated rows back to a partition‑ed Hive table `mnaas.traffic_details_raw_daily_with_no_dups_summation`. The job is invoked from the nightly orchestration layer (e.g., via a Bash wrapper that runs `spark-submit`) and forms the “summation” step of the traffic‑no‑dups pipeline.

---

### 1. Important Functions / Classes  

| Symbol | Responsibility |
|--------|-----------------|
| **`traffic_nodups_summation(start_date, end_date)`** | Core routine that builds Spark SQL queries, performs three independent aggregations (Data, Voice, SMS), de‑duplicates rows using `row_number()`, trims several textual columns, adds a `partition_month` column, unions the three result sets, left‑joins to `mnaas.org_details` for customer name, and finally writes the output to the target Hive table. |
| **`if __name__ == '__main__'` block** | Parses two positional arguments (`start_date`, `end_date`), creates a `SparkSession` named *traffic_nodups_summation*, and calls the core routine. No custom classes are defined. |

---

### 2. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Command‑line arguments: `start_date` (YYYY‑MM‑DD) and `end_date` (YYYY‑MM‑DD). <br> - Hive tables: <br>   • `mnaas.traffic_details_raw_daily_with_no_dups` (source, partitioned by `partition_date`). <br>   • `mnaas.org_details` (lookup for `orgname`). |
| **Outputs** | - Hive table `mnaas.traffic_details_raw_daily_with_no_dups_summation` written in **append** mode, partitioned by `partition_month` (derived from `partition_date`). |
| **Side Effects** | - Creates temporary Spark SQL views (`t1` … `t7`). <br> - May generate intermediate shuffle data; relies on cluster storage for spill. |
| **Assumptions** | - The source table contains the columns listed in the `columns` array (exact order required for the final `select`). <br> - `calltype` values are exactly `'Data'`, `'Voice'`, `'SMS'`. <br> - `sim` uniquely identifies a device for the aggregation window. <br> - Hive metastore is reachable and the target table already exists with compatible schema. <br> - Spark configuration (dynamic partitioning, etc.) is accepted by the cluster. |

---

### 3. Integration with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **Bash orchestration wrappers** (e.g., `runTrafficSummation.sh` or a generic `spark-submit` launcher) | Invokes this script with `spark-submit traffic_nodups_summation.py $START $END`. Handles scheduling, logging, and status‑file management (similar to other `run*.sh` scripts in the repo). |
| **`traffic_details_raw_daily_with_no_dups` producer** (e.g., `traffic_nodups_extractor.py` or a Hive load job) | Supplies the raw de‑duplicated data that this job reads. Must run **before** the summation step. |
| **Downstream reporting / analytics jobs** (e.g., `traffic_aggr_adhoc_refresh.sh`, `product_status_report.sh`) | Consume the `*_summation` table for aggregated metrics, dashboards, or SLA checks. |
| **Cluster resource manager** (YARN / Kubernetes) | Provides Spark executors; job relies on proper queue/priority configuration defined outside this script. |
| **Monitoring / alerting** (e.g., Airflow, cron, custom watchdog) | Detects non‑zero exit codes or missing output partitions and raises tickets/emails. |

---

### 4. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect date range** (e.g., start > end) leads to empty output or duplicate processing. | Data gaps / duplicated rows in downstream tables. | Validate arguments early; abort with clear error if `start_date > end_date`. |
| **Schema drift** (source table gains/loses columns). | Job fails at `select(*columns)` or writes malformed rows. | Add a schema‑validation step (compare `columns` list with `spark.table(...).schema`). |
| **Large shuffle volume** (high‑volume day). | Out‑of‑memory executor failures, long runtimes. | Tune Spark configs (shuffle partitions, executor memory) via external config; consider incremental processing per partition_date. |
| **Missing lookup rows** in `org_details`. | `customername` becomes null, downstream reports may be incomplete. | Log count of unmatched `tcl_secs_id`; optionally fallback to a default value. |
| **Concurrent runs** (multiple instances for overlapping dates). | Duplicate rows in target table. | Enforce single‑instance lock at orchestration level (status file / PID lock) – already used by other scripts. |
| **Append mode without idempotency** – re‑running the same window creates duplicate rows. | Data inflation. | Either truncate target partitions before write or switch to `overwrite` on the specific `partition_month`. |

---

### 5. Example: Running & Debugging  

**Typical production launch (via Bash wrapper):**  

```bash
# Assume $START and $END are YYYY-MM-DD strings
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --conf spark.sql.shuffle.partitions=200 \
  --conf spark.yarn.maxAppAttempts=1 \
  move-mediation-scripts/bin/traffic_nodups_summation.py $START $END
```

**Ad‑hoc local test (small date range):**  

```bash
export PYSPARK_PYTHON=python3
spark-submit --master local[4] traffic_nodups_summation.py 2024-10-01 2024-10-01
```

**Debugging tips:**  

1. **View temporary views** – after the job starts, attach a Spark shell to the same application ID and run `spark.sql("show tables").show()` to inspect `t1`‑`t7`.  
2. **Row‑level inspection** – `spark.sql("select * from t7 limit 10").show(truncate=False)`.  
3. **Log counts** – add `print(df1.count())` or `df1.groupBy("calltype").count().show()` before the final write to verify aggregation.  
4. **Check partitions** – after completion, run `SHOW PARTITIONS mnaas.traffic_details_raw_daily_with_no_dups_summation;` to confirm expected `partition_month` values.  

---

### 6. External Configuration / Environment Variables  

| Variable / File | Usage |
|-----------------|-------|
| **Spark configuration** (e.g., `spark.sql.dynamic.partition`, `spark.sql.debug.maxToStringFields`) – set inside the script. | Controls Hive dynamic partitioning and debug output length. |
| **Cluster‑wide configs** (YARN queue, executor memory, shuffle partitions) – passed via `spark-submit --conf` or environment files (`spark-defaults.conf`). | Not hard‑coded; must be supplied by the orchestration wrapper. |
| **Hive metastore connection** – relies on default Hive settings (`hive.metastore.uris`) from the Spark environment. | No explicit env vars in the script. |
| **Logging / monitoring hooks** – external wrappers may export `JOB_ID`, `RUN_DATE`, etc., but the script itself does not read them. | Ensure wrappers propagate any needed context. |

---

### 7. Suggested TODO / Improvements  

1. **Argument validation & schema check** – add a guard that verifies `start_date <= end_date` and that all columns in the `columns` list exist in the source table; fail fast with a clear message.  
2. **Idempotent write strategy** – replace `mode("append")` with a partition‑aware `overwrite` (or a pre‑write `DELETE FROM … WHERE partition_month = …`) to prevent duplicate rows when a job is retried for the same window.  

--- 

*End of documentation.*