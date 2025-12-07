**High‑Level Documentation – `move-mediation-scripts/bin/move_table_compute_stats.sh`**

---

### 1. Purpose (One‑paragraph summary)

`move_table_compute_stats.sh` is a scheduled Bash utility that iterates over a configurable list of MOVE‑schema tables, determines whether each table’s statistics are stale based on a per‑table refresh interval, and triggers an Impala **`COMPUTE INCREMENTAL STATS`** command for tables that need updating. After a successful compute, the script rewrites the table‑list file with the new “last‑computed” timestamp, ensuring future runs only recompute when required. All activity is logged to a dedicated log file for operational visibility.

---

### 2. Key Components & Responsibilities

| Component | Type | Responsibility |
|-----------|------|----------------|
| **`move_table_compute_stats.properties`** | External properties file | Supplies runtime variables: `MOVE_TABLE_COMPUTE_STATS_LOG`, `TABLE_LIST`, `IMPALAD_HOST`, etc. |
| **Main loop** (`for table in \`cat $TABLE_LIST\``) | Bash loop | Reads each line of `$TABLE_LIST` (format `table_name|last_stat_time|refresh_interval`). |
| **Timestamp logic** (`diff_stat_time`, `REFRESH_TIME_INTERVAL`) | Bash arithmetic | Determines if the elapsed time since the last stats compute exceeds the configured interval. |
| **Impala invocation** (`impala-shell -i $IMPALAD_HOST -q "compute INCREMENTAL stats $table_name"`) | External command | Executes the incremental stats computation on the target Impala daemon. |
| **Success handling** (`if [ $? -eq 0 ]`) | Bash conditional | On success, updates the `$TABLE_LIST` entry with the new timestamp (`sed -i …`). |
| **Logging** (`logger -s …`) | Bash built‑in + syslog | Writes start/completion messages (including timestamps) to both stdout and `${MOVE_TABLE_COMPUTE_STATS_LOG}`. |
| **Exit redirection** (`exec 2>&1`, `exec 1>>${MOVE_TABLE_COMPUTE_STATS_LOG}`) | Bash I/O redirection | Routes all script output to the designated log file. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `move_table_compute_stats.properties` (defines all env vars).<br>• `$TABLE_LIST` – plain‑text file, one line per table: `table_name|last_stat_time|refresh_interval`.<br>• Environment variables referenced indirectly via the properties file: `MOVE_TABLE_COMPUTE_STATS_LOG`, `IMPALAD_HOST`. |
| **Outputs** | • Log entries appended to `${MOVE_TABLE_COMPUTE_STATS_LOG}`.<br>• Updated `$TABLE_LIST` with refreshed timestamps (in‑place edit). |
| **Side‑effects** | • Calls `impala-shell` → updates Impala’s internal statistics for the table.<br>• Modifies `$TABLE_LIST` file (writes new timestamps). |
| **Assumptions** | • The properties file exists and is readable by the script user.<br>• `$TABLE_LIST` exists, is writable, and follows the expected pipe‑delimited format.<br>• `impala-shell` is installed, reachable, and the `$IMPALAD_HOST` is up and authorized for the user.<br>• The script runs with sufficient OS permissions to edit `$TABLE_LIST` and write to the log file.<br>• No concurrent executions (no lock file) – otherwise race conditions on `$TABLE_LIST` may occur. |

---

### 4. Interaction with Other Scripts / System Components

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_tbl_load_generic.sh` / `mnaas_tbl_load.sh`** | These loading scripts likely populate the MOVE tables; after load they may trigger or schedule `move_table_compute_stats.sh` to refresh stats. |
| **`mnaas_seq_check_for_feed.sh`** | May generate or update `$TABLE_LIST` with new tables to monitor; the compute‑stats script consumes that list. |
| **Scheduler (e.g., cron, Oozie, Airflow)** | The script is typically invoked on a regular cadence (e.g., hourly) to keep stats fresh. |
| **Impala Cluster** | The target of the `impala-shell` command; stats are stored in Impala’s catalog. |
| **Logging infrastructure** | `logger -s` writes to syslog; the log file is later harvested by monitoring tools (Splunk, ELK). |
| **Properties repository** | All scripts under `move-mediation-scripts/bin/` source a common properties directory (`MNAAS_Property_Files`). Changes to that directory affect this script. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed `$TABLE_LIST`** | No tables processed; script may exit with errors. | Validate file existence and format at start; abort with clear error code. |
| **Impala host unreachable / authentication failure** | Stats not computed; timestamps not updated → stale stats. | Add retry logic with exponential back‑off; alert on non‑zero exit status. |
| **Concurrent executions** (e.g., overlapping cron runs) | Race condition on `$TABLE_LIST` leading to corrupted timestamps. | Implement a lock file (`flock`) around the whole script or around the `sed` operation. |
| **`sed -i` failure (permission or disk full)** | Table list not updated → repeated unnecessary compute attempts. | Check exit status of `sed`; log failure; optionally send alert. |
| **Unbounded log growth** | Disk exhaustion on the log file. | Rotate logs via logrotate or size‑based truncation; include log‑size monitoring. |
| **Incorrect refresh interval** (e.g., zero or negative) | Either never compute or compute on every run, causing load spikes. | Validate `REFRESH_TIME_INTERVAL` > 0 before comparison. |

---

### 6. Typical Execution / Debugging Workflow

1. **Run manually (dry‑run)**  
   ```bash
   set -x   # enable bash tracing
   ./move_table_compute_stats.sh
   ```
   Observe each command printed to stdout and verify log entries.

2. **Check prerequisites**  
   ```bash
   test -f /app/hadoop_users/MNAAS/MNAAS_Property_Files/move_table_compute_stats.properties && echo OK
   test -f $TABLE_LIST && echo "Table list present"
   impala-shell -i $IMPALAD_HOST -q "show tables;"   # confirm connectivity
   ```

3. **Validate `$TABLE_LIST` format**  
   ```bash
   awk -F'|' '{if (NF!=3) print "BAD:", $0}' $TABLE_LIST
   ```

4. **Inspect log after run**  
   ```bash
   tail -n 20 ${MOVE_TABLE_COMPUTE_STATS_LOG}
   ```

5. **Confirm stats were updated** (example for a specific table)  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "show table stats <table_name>;"
   ```

6. **Troubleshoot failures**  
   * Non‑zero exit from `impala-shell` → check Impala logs, network, authentication.  
   * `sed` error → verify write permission on `$TABLE_LIST`.  

---

### 7. External Configuration & Environment Variables

| Variable (defined in `.properties`) | Description |
|------------------------------------|-------------|
| `MOVE_TABLE_COMPUTE_STATS_LOG` | Full path to the log file where script output is appended. |
| `TABLE_LIST` | Path to the pipe‑delimited file containing `table_name|last_stat_time|refresh_interval`. |
| `IMPALAD_HOST` | Hostname or IP of the Impala daemon to which `impala-shell` connects. |
| *(Potential others)* | The properties file may also define `HADOOP_CONF_DIR`, `JAVA_HOME`, etc., inherited from the broader MOVE environment. |

---

### 8. Suggested Improvements (TODO)

1. **Add robust locking** – Use `flock` or a PID file to prevent simultaneous runs that could corrupt `$TABLE_LIST`.  
2. **Centralize error handling** – Wrap the Impala call and the `sed` update in a function that logs failures, retries a configurable number of times, and optionally sends an alert (e.g., via email or monitoring webhook).  

--- 

*End of documentation.*