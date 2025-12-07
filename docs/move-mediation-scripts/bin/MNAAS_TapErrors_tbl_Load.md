**MNAAS_TapErrors_tbl_Load.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_TapErrors_tbl_Load.sh` is the daily ingestion pipeline for the *TapErrors* data set. It discovers newly‑arrived CSV files in the staging area, validates their naming and content, backs them up, strips headers, merges them into a single file, copies the result to HDFS, loads the data into a temporary Hive/Impala table via a Java loader, and finally inserts the cleaned rows into the production *TapErrors* raw table. The script maintains a process‑status file to allow safe restarts, logs every step, and raises an SDP ticket with email notification on failure.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Scans the staging directory, validates file‑name pattern and that the file is readable text. Moves good files to the *intermediate* directory, bad files to a *rejected* folder, and empties to an *empty‑file* folder. Updates process‑status flag to **1**. |
| **MNAAS_cp_files_to_backup** | Copies all intermediate files to a daily backup directory. Updates status flag to **2**. |
| **MNAAS_rm_header_from_files** | Removes the CSV header from each intermediate file, replaces the first comma with a pipe (`|`) to preserve embedded commas, and prefixes each line with the original filename. Writes results to a *header‑removed* folder. Updates status flag to **3**. |
| **MNAAS_merge_files** | Concatenates all header‑removed files into a single merge file (`$Merge_TapErrors_file`) and deletes the source files. Updates status flag to **4**. |
| **MNAAS_mv_files_to_hdfs** | Clears the target HDFS raw‑load directory, then copies the merged files from the local *header‑removed* folder to HDFS. Updates status flag to **5**. |
| **MNAAS_load_files_into_temp_table** | Invokes a Java class (`$Load_nonpart_table`) to bulk‑load the HDFS files into a temporary Hive/Impala table (`$TapErrors_daily_inter_tblname`). Refreshes the Impala metadata. Cleans up local intermediate folders. Updates status flag to **6**. |
| **MNAAS_insert_into_raw_table** | Calls a second Java class (`$Insert_Part_Daily_table`) to insert data from the temporary table into the final raw table (`$TapErrors_daily_tblname`). Refreshes Impala metadata. Updates status flag to **7**. |
| **terminateCron_successful_completion** | Writes “Success” into the process‑status file, logs completion, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a pre‑formatted mail to the support mailbox and creates an SDP ticket (once per run) when the script fails. |
| **Main block** | Implements a lock‑file mechanism using the process‑status file, decides which functions to run based on the stored flag (allowing restart from any step), and orchestrates the full pipeline. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Input files** | CSV files in `$MNAASMainStagingDirDaily_v1/$TapErrors_extn` matching pattern `^[a-zA-Z]+_[0-9]+_[a-zA-Z]+_[a-zA-Z0-9]+_[0-9]+\.csv$`. |
| **Intermediate directories** | <ul><li>`$MNAASInterFilePath_Daily_TapErrors` – validated files</li><li>`$MNAASInterFilePath_Daily_TapErrors_rm_header` – header‑removed files</li><li>`$MNAAS_TapErrors_Error_FileName` – list of files failing the “text” test</li></ul> |
| **Backup** | `$Daily_TapErrors_BackupDir` receives a copy of every validated file. |
| **Rejected/Empty** | `$MNAASRejectedFilePath` and `$EmptyFileDir` receive malformed or empty files. |
| **Merged file** | `$Merge_TapErrors_file` (single aggregated CSV). |
| **HDFS target** | `$MNAAS_Daily_Rawtablesload_TapErrors_PathName` (cleared before each run). |
| **Hive/Impala tables** | Temporary table `$TapErrors_daily_inter_tblname`; final raw table `$TapErrors_daily_tblname`. |
| **Process‑status file** | `$MNAAS_Daily_TapErrors_Load_Aggr_ProcessStatusFileName` – holds flags, PID, timestamps, job status, email‑sent flag. |
| **Log file** | `$MNAAS_DailyTapErrorsLoadAggrLogPath` (appended throughout). |
| **Side effects** | <ul><li>Creates/updates files on local FS and HDFS.</li><li>Runs Java JARs that interact with Hive/Impala.</li><li>Sends email & creates SDP ticket on failure.</li></ul> |
| **Assumptions** | <ul><li>All required environment variables are defined in the sourced properties file.</li><li>Hadoop, Hive, Impala services are reachable.</li><li>Java JARs (`$Load_nonpart_table`, `$Insert_Part_Daily_table`) are present and compatible.</li><li>Only one instance runs at a time (PID lock).</li></ul> |

---

### 4. Integration Points (how this script connects to other components)

| Component | Connection Detail |
|-----------|-------------------|
| **Up‑stream file producers** | Other mediation scripts (e.g., `MNAAS_Sqoop_*` jobs) generate the raw TapErrors CSVs and drop them into the staging directory `$MNAASMainStagingDirDaily_v1/$TapErrors_extn`. |
| **Backup & retention** | The daily backup directory is consumed by archival/retention processes (e.g., a separate cleanup cron). |
| **HDFS raw‑load area** | The path `$MNAAS_Daily_Rawtablesload_TapErrors_PathName` is read by downstream analytics jobs that query the *TapErrors* raw table. |
| **Hive/Impala** | Temporary and raw tables are part of the data‑warehouse schema used by reporting dashboards and other ETL pipelines. |
| **Process‑status file** | Shared with any “resume” or monitoring script that may query the flag values (e.g., a generic health‑check daemon). |
| **Alerting** | `email_and_SDP_ticket_triggering_step` integrates with the corporate SDP ticketing system and the support mailbox (`insdp@tatacommunications.com`). |
| **Logging/Monitoring** | Log file is harvested by log‑aggregation tools (Splunk/ELK) for operational dashboards. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect file naming or malformed CSV** | Files may be silently rejected, causing data loss for that day. | Add a pre‑run audit that lists expected file names; send a warning if count < threshold. |
| **`hadoop fs -rm` removes unexpected data** | If the target HDFS directory contains files from a previous successful run, they will be deleted. | Use `-trash` option or move to a temporary “to‑delete” folder before removal; add a safety check on file count. |
| **Java loader JAR failure (exit code ≠ 0)** | Pipeline stops, raw table not refreshed. | Capture stdout/stderr of the Java call, archive them, and implement a retry with exponential back‑off. |
| **Concurrent executions** | Two instances could corrupt the process‑status file or duplicate loads. | The PID lock already prevents this; ensure the lock file is atomically updated and consider using `flock`. |
| **Disk‑space exhaustion (local or HDFS)** | Copy/merge steps may fail, leading to incomplete loads. | Monitor free space before each step; abort early with a clear log message. |
| **Missing/changed environment variables** | Script will exit with cryptic errors. | Validate all required variables at start (e.g., `[[ -z $var ]] && { logger "Missing $var"; exit 1; }`). |
| **Email/SDP ticket spam** | Repeated failures could flood the ticketing system. | Guard against repeated ticket creation (already done via flag) and add a rate‑limit. |

---

### 6. Running / Debugging the Script  

| Action | Command / Steps |
|--------|-----------------|
| **Standard execution** | The script is scheduled via cron (e.g., `0 2 * * * /app/hadoop_users/MNAAS/.../MNAAS_TapErrors_tbl_Load.sh`). Ensure the properties file exists and is readable. |
| **Manual run (debug mode)** | ```bash\nset -x   # enable tracing\n/app/hadoop_users/MNAAS/.../MNAAS_TapErrors_tbl_Load.sh\n``` |
| **Force start from a specific step** | Edit the process‑status file (`$MNAAS_Daily_TapErrors_Load_Aggr_ProcessStatusFileName`) and set `MNAAS_Daily_ProcessStatusFlag` to the desired step number (1‑7) before invoking the script. |
| **Check logs** | Tail the log: `tail -f $MNAAS_DailyTapErrorsLoadAggrLogPath`. |
| **Inspect intermediate files** | Verify contents of `$MNAASInterFilePath_Daily_TapErrors_rm_header` and `$Merge_TapErrors_file` if the merge step fails. |
| **Validate Java loader** | Run the Java command manually with `-verbose:class` to see classpath issues. |
| **Verify HDFS copy** | `hadoop fs -ls $MNAAS_Daily_Rawtablesload_TapErrors_PathName`. |
| **Check process lock** | `cat $MNAAS_Daily_TapErrors_Load_Aggr_ProcessStatusFileName` to see stored PID and flag. |

---

### 7. External Configuration, Environment Variables & Files  

| Item | Source | Usage |
|------|--------|-------|
| **`MNAAS_TapErrors_tbl_Load.properties`** | Sourced at top of script (`. /app/hadoop_users/MNAAS/.../MNAAS_TapErrors_tbl_Load.properties`) | Defines all `$MNAAS*` variables: staging dir, file extensions, backup dirs, HDFS paths, Hive/Impala connection details, Java class names, log paths, email recipients, etc. |
| **`$setparameter`** | Likely a function/alias defined elsewhere to export common env vars (e.g., `JAVA_HOME`, `HADOOP_CONF_DIR`). | Must be available in the runtime environment. |
| **`$CLASSPATHVAR`, `$Generic_Jar_Names`, `$MNAAS_Main_JarPath`** | Populated by the properties file or a global env script. | Provide the classpath for the Java loader JARs. |
| **`$SDP_ticket_from_email`, `$MOVE_DEV_TEAM`** | Email configuration variables. | Used when sending failure notifications. |
| **`$MNAAS_Daily_TapErrors_Load_Aggr_ProcessStatusFileName`** | Path to a plain‑text status file. | Stores flags, PID, timestamps, job status, and email‑sent flag. |
| **`$MNAAS_DailyTapErrorsLoadAggrLogPath`** | Log file location. | All `logger` output is appended here. |
| **`$Merge_TapErrors_file`** | Temporary merged CSV file. | Consumed by HDFS copy step. |
| **`$TapErrors_extn`** | File extension (e.g., `.csv`). | Used in `ls`/`find` patterns. |
| **`$IMPALAD_HOST`, `$HIVE_HOST`, `$HIVE_JDBC_PORT`** | Hive/Impala connection details. | Required for `impala-shell` refresh commands and Java loader. |

If any of these variables are missing, the script will abort with a non‑zero exit code; therefore they must be validated before production runs.

---

### 8. Suggested Improvements (TODO)

1. **Atomic status‑file updates** – Replace the series of `sed -i` commands with a single `awk` or temporary‑file rewrite to avoid race conditions if the script is ever invoked concurrently (e.g., by a manual run while cron is active).

2. **Safe HDFS delete** – Use `hadoop fs -mv … .trash/` or enable the Hadoop trash feature before removing files in `$MNAAS_Daily_Rawtablesload_TapErrors_PathName`. This prevents accidental loss of data from a previous successful load.  

(Additional enhancements could include parameterizing the “head -$No_of_files_to_process” limit, adding retry logic for the Java loaders, and emitting Prometheus metrics for each step.)

---