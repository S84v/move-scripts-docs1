**MNAAS_Telena_SNG_files_Stat_process.sh – High‑Level Documentation**  

---

### 1. Purpose (one‑paragraph summary)  
This script is the nightly “Telena SNG statistics” loader. For a given processing date (default = yesterday) it checks for the presence of the CSV file `SNG_Stat_filename_telena_<date>.csv`, copies it into HDFS, loads it into a Hive staging table, then overwrites the partitioned production table with the new data and refreshes the corresponding Impala view. Throughout the run it updates a shared *process‑status* file, writes detailed logs, and, on failure in production, raises an SDP ticket and sends an email notification. The script is designed to be invoked by a cron job and to guard against concurrent executions.

---

### 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **load_backup_files_records_to_temp_table** | *Step 1*: Waits (polls every 5 min) for the expected CSV file, removes any previous HDFS copy, copies the file to HDFS, truncates the Hive staging table, and loads the CSV into that staging table. Updates the status file flag to **1** (loading). |
| **insert_data_to_mainTable** | *Step 2*: Sets status flag to **2**, inserts data from the staging table into the partitioned production table (`$telena_sng_file_record_count_tblname`) using dynamic partitions, then runs an Impala refresh query (`$telena_sng_file_record_count_refresh`). |
| **terminateCron_successful_completion** | Resets the status file flags to *idle* (`0`), marks job status **Success**, records run‑time, writes final log entries and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, optionally triggers `email_and_SDP_ticket_triggering_step` when `ENV_MODE=PROD`, writes end‑time and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Checks the “email‑sent” flag; if not yet sent, composes a minimal ticket‑style email (to Cloudera Support and a designated recipient) and updates the status file to indicate that an SDP ticket/email has been created. |
| **Main program block** | Prevents parallel runs by checking the PID stored in the status file, decides which steps to execute based on the current flag value (0/1 → load + insert, 2 → insert only), and drives the overall flow. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input parameters** | Optional positional argument `prev_date` (format `YYYYMMDD`). If omitted, defaults to yesterday’s date. |
| **External files** | • `${MNAAS_Telena_files_backup_ProcessStatusFile}` – a plain‑text key/value file used for inter‑script coordination and monitoring.<br>• `${MNAAS_Telena_files_backup_ProcessStatusFile}` is also read/written for PID, flags, timestamps, email‑sent flag, etc.<br>• `${MNAAS_Telena_files_backup_ProcessStatusFile}` path is defined in the sourced properties file. |
| **Data files** | • `${MNAAS_Customer_SNG_Stat_file_path}/SNG_Stat_filename_telena_${prev_date}.csv` – the source CSV produced by the Telena system.<br>• HDFS target directory `${MNAAS_sng_filename_hdfs_Telena}`. |
| **Hive/Impala objects** | • Staging table: `$dbname.$telena_sng_file_record_count_inter_tblname` (truncated before each load).<br>• Production table: `$dbname.$telena_sng_file_record_count_tblname` (partitioned on `file_date`).<br>• Impala refresh query stored in `$telena_sng_file_record_count_refresh`. |
| **Outputs** | • Populated production Hive table (partition for `prev_date`).<br>• Updated status file (flags, PID, timestamps, job status).<br>• Log file `${MNAAS_backup_files_logpath_Telena}` (appended). |
| **Side‑effects** | • Deletes any existing HDFS files under the target directory before copy.<br>• May delete the source CSV (commented out line).<br>• Sends email / creates SDP ticket on failure (only in PROD). |
| **Assumptions** | • All `$MNAAS_*` variables are correctly defined in the sourced `.properties` file.<br>• Hadoop, Hive, and Impala CLI tools are available on the host and reachable (`$IMPALAD_HOST`).<br>• The CSV file is well‑formed for Hive `LOAD DATA` (no header, proper delimiters).<br>• The status file is writable by the script user and is the single source of truth for concurrency control. |
| **External services** | • HDFS (via `hadoop fs`).<br>• Hive Metastore (via `hive -e`).<br>• Impala daemon (`impala-shell`).<br>• Mail system (`mailx`).<br>• SDP ticketing (implicit – email triggers ticket creation). |

---

### 4. Integration with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **MNAAS_Telena_SNG_files_Stat_process.properties** | Provides all `$MNAAS_*` variables (paths, table names, email flags, environment mode, etc.). |
| **Other “MNAAS_*_backup_files” scripts** | Share the same `${MNAAS_Telena_files_backup_ProcessStatusFile}` to coordinate daily batch windows and avoid overlapping runs. |
| **Cron scheduler** | Typically invoked nightly (e.g., `0 2 * * * /path/MNAAS_Telena_SNG_files_Stat_process.sh`). |
| **Monitoring / Alerting** | Log entries are written to `${MNAAS_backup_files_logpath_Telena}`; external log‑aggregation tools (Splunk, ELK) may ingest these. |
| **SDP ticketing system** | Triggered indirectly via the email sent by `email_and_SDP_ticket_triggering_step`. |
| **Downstream reporting jobs** | Consume the partitioned Hive table `$telena_sng_file_record_count_tblname`; they assume the data is refreshed before their own execution. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed CSV** – script loops waiting 5 min, may stall indefinitely. | Data pipeline delay, resource waste. | Add a maximum retry count / timeout; raise alert if file never appears. |
| **Concurrent executions** – PID check may be bypassed if status file is corrupted. | Duplicate loads, data corruption. | Store PID in a lock file with `flock`; verify process existence more robustly. |
| **Hive/Impala failures** – non‑zero exit leads to immediate termination, but temporary tables may remain. | Partial data load, need manual cleanup. | Implement cleanup steps (drop temp table, remove HDFS files) in a `finally` block. |
| **Hard‑coded paths & sed edits** – any change in status‑file format breaks the script. | Script stops, downstream jobs fail. | Move status handling to a small helper utility (e.g., `jq` on JSON) or use a key‑value store. |
| **Email/SDP spam** – repeated failures could flood inboxes. | Alert fatigue. | Add rate‑limiting (e.g., only send once per hour) and include a unique ticket ID. |
| **No explicit error handling for `hadoop fs -rm`** – if removal fails, stale files may cause load errors. | Load failure. | Check exit status of each HDFS command and log/abort on error. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Standard execution** | `./MNAAS_Telena_SNG_files_Stat_process.sh` (runs for yesterday) or `./MNAAS_Telena_SNG_files_Stat_process.sh 20231201` (specific date). |
| **Enable verbose tracing** | Uncomment `set -x` near the top of the script (currently commented). |
| **Check status before run** | `cat $MNAAS_Telena_files_backup_ProcessStatusFile` – verify `MNAAS_Daily_ProcessStatusFlag` is `0`. |
| **Force re‑run** | If a previous run left flag `2`, you can manually set it to `0` (or delete the status file) before invoking. |
| **Inspect logs** | Tail the log file: `tail -f $MNAAS_backup_files_logpath_Telena`. |
| **Validate Hive tables** | After a successful run, run `hive -e "SELECT COUNT(*) FROM $dbname.$telena_sng_file_record_count_tblname WHERE file_date='${prev_date}'"` to confirm row count. |
| **Debug HDFS copy** | Manually execute the `hadoop fs -copyFromLocal` command shown in the script to see any permission errors. |
| **Simulate failure** | Temporarily set `ENV_MODE=PROD` and force a Hive error (e.g., rename the staging table) to test email/SDP path. |
| **Process lock inspection** | `ps -fp $(grep MNAAS_Script_Process_Id $MNAAS_Telena_files_backup_ProcessStatusFile | cut -d'=' -f2)` to see the PID that holds the lock. |

---

### 7. External Configuration / Environment Variables  

| Variable (populated in `.properties`) | Meaning |
|---------------------------------------|---------|
| `MNAAS_Telena_files_backup_ProcessStatusFile` | Path to the shared status/key‑value file. |
| `MNAAS_backup_files_logpath_Telena` | Directory/file where script logs are appended. |
| `MNAAS_Customer_SNG_Stat_file_path` | Local directory containing the incoming CSV files. |
| `MNAAS_sng_filename_hdfs_Telena` | HDFS target directory for the CSV copy. |
| `dbname` | Hive database name. |
| `telena_sng_file_record_count_inter_tblname` | Hive staging table name. |
| `telena_sng_file_record_count_tblname` | Production Hive table (partitioned). |
| `telena_sng_file_record_count_refresh` | Impala `REFRESH` statement (or other DDL) to make new data visible. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. |
| `MNAAS_telena_SNG_files_Stat_Scriptname` | Human‑readable script identifier used in logs/emails. |
| `ENV_MODE` | Execution environment (`PROD`, `UAT`, etc.) – controls ticket/email generation. |
| `MNAAS_email_sdp_created` | Flag stored in the status file to avoid duplicate tickets. |
| `MNAAS_job_status` | Overall job status (`Success`/`Failure`). |
| `MNAAS_job_ran_time` | Timestamp of the last run (written to status file). |

If any of these variables are missing or point to non‑existent locations, the script will abort with a non‑zero exit code.

---

### 8. Suggested TODO / Improvements  

1. **Replace ad‑hoc `sed` status‑file edits with a small key‑value utility** (e.g., a Bash function that reads/writes a JSON file via `jq`). This will make the status handling more robust and easier to extend.  
2. **Add a configurable maximum wait‑time for the input CSV** (e.g., `MAX_WAIT_MINUTES=60`). After the timeout, the script should fail gracefully and raise an alert rather than looping forever.  

--- 

*End of documentation.*