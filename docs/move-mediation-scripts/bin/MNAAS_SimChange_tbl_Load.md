**MNAAS_SimChange_tbl_Load.sh – High‑Level Documentation**  

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_SimChange_tbl_Load.sh` is the daily “Traffic Details” ingestion pipeline for the MNAAS mediation layer. It validates, stages, de‑duplicates, and merges raw CDR‑style CSV files delivered to the daily staging area, copies the consolidated data to HDFS, loads it into a Hive temporary table, then moves the rows into the final raw and reject tables. After the load it records the partition dates, triggers an hourly aggregation script and updates a process‑status flag file so that the pipeline can be resumed from the last successful step after a failure or restart. The script is orchestrated by a cron job and integrates with Hadoop/HDFS, Hive/Impala, Java ETL JARs, and the internal ticket‑ing/notification system.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **MNAAS_files_pre_validation** | Sets flag = 1, clears intermediate dir, selects the first *N* files from the staging area, validates filename pattern and that the file is readable text. Moves good files to the intermediate “Traffic Details” dir, bad files to a reject dir, empties are moved to an *EmptyFileDir*. |
| **MNAAS_cp_files_to_backup** | Flag = 2. Copies the intermediate files to a daily backup directory. |
| **MNAAS_rm_header_from_files** | Flag = 3. Strips the first line (header) from every intermediate CSV. |
| **MNAAS_append_filename_to_start_of_records** | Flag = 4. Prefixes each record with the source filename followed by a semicolon. |
| **MNAAS_move_files_to_another_temp_dir** | Flag = 5. Moves the prefixed files to a temporary “with‑dups” directory, cleaning any previous content. |
| **MNAAS_remove_duplicates_in_the_files** | Flag = 6. Removes duplicate lines per file using `awk '!a[$0]++'`, writes to a “without‑dups” dir, logs line counts before/after. |
| **MNAAS_move_nodups_files_to_inter_dir** | Flag = 7. Copies de‑duplicated files back to the main intermediate dir and also to a merge‑ready dir. |
| **MNAAS_mergefile_to_separate_dir** | Flag = 8. If a lock file (`$MergeLockName`) exists, concatenates all merge‑ready files into a single *Mergeinput* file, then cleans the lock and the merge‑ready dir. If the lock is absent (outside the scheduled hour) the script exits cleanly. |
| **MNAAS_load_files_into_temp_table** | Flag = 9. Copies the merged file to HDFS, then runs a Java JAR (`$Load_nonpart_table`) to bulk‑load into a Hive temporary table. Refreshes the table via Impala. |
| **MNAAS_insert_into_raw_table** | Flag = 10. Executes a Java JAR (`$Insert_Part_Daily_table`) to move rows from the temp table to the final raw table, refreshes via Impala, and cleans intermediate dirs. |
| **MNAAS_insert_into_reject_table** | Flag = 11. Similar to the raw insert but targets the reject table (`$Insert_Part_Daily_Reject_table`). |
| **MNAAS_save_partitions** | Flag = 12. Queries Hive for distinct `calldate` partitions (excluding test customers) and writes them to `$MNAAS_Traffic_Partitions_Hourly_FileName`. |
| **MNAAS_hourly_aggr** | Flag = 13. Calls an external shell script (`$MNAASDailyTrafficDetails_Hourly_Aggr_SriptName`) passing the current hour/minute to perform hourly aggregations. |
| **terminateCron_successful_completion** | Resets flag = 0, marks job status *Success*, writes run‑time, logs completion and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a templated email via `mailx` to the support team and marks the ticket‑created flag to avoid duplicate alerts. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Primary Input Files** | CSV files placed in `$MNAASMainStagingDirDaily_afr_seq_check/$Traffic_Details_extn` (naming pattern `^[a-zA-Z]+_[0-9]+_[a-zA-Z]+_[0-9]+_[0-9]+\.csv$`). |
| **Intermediate Directories** | `$MNAASInterFilePath_Daily_Traffic_Details`, `$MNAASInterFilePath_Daily_Traffic_Details_MergeFiles`, `$MNASS_Intermediatefiles_removedups_withdups_filepath`, `$MNASS_Intermediatefiles_removedups_withoutdups_filepath`, `$Daily_TrafficDetails_BackupDir`, `$MNAASRejectedFilePath`, `$EmptyFileDir`. |
| **HDFS Destination** | `$MNAAS_Daily_Rawtablesload_TrafficDetails_PathName` (cleared before each run). |
| **Hive/Impala Tables** | Temp table `$traffic_details_daily_inter_tblname`, raw table `$traffic_details_daily_tblname`, reject table `$traffic_details_daily_reject_tblname`. |
| **Process‑status File** | `$MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName` – holds flags, process name, PID, job status, timestamps, email‑ticket flag. |
| **Log File** | `$MNAAS_DailyTrafficDetailsLoadAggrLogPath` (stderr redirected here). |
| **External Services** | Hadoop CLI (`hadoop fs`), Hive CLI, Impala CLI, Java runtime (JARs referenced by `$Load_nonpart_table`, `$Insert_Part_Daily_table`, `$Insert_Part_Daily_Reject_table`), `mailx` for notifications, system logger (`logger`). |
| **Side Effects** | - Files moved/renamed across several directories.<br>- HDFS data overwritten each run.<br>- Hive tables populated/updated.<br>- Email/SLA ticket may be generated on failure.<br>- Process‑status file is mutated for checkpointing. |
| **Assumptions** | - All directory paths and the property file exist and are writable by the script user.<br>- Hadoop, Hive, Impala CLIs and required Java JARs are installed and reachable via `$CLASSPATHVAR`.<br>- Sufficient disk space for backup and intermediate files.<br>- The lock file `$MergeLockName` is created by a scheduler to allow merging only during the designated hour.<br>- `mailx` is configured for outbound mail. |

---

### 4. Integration Points (how it connects to other scripts/components)  

| Component | Relationship |
|-----------|--------------|
| **Up‑stream** | Other “MNAAS_*_Load.sh” scripts (e.g., `MNAAS_TrafficDetails_tbl_Load.sh`, `MNAAS_SimChange_seq_check.sh`) drop raw CSV files into the staging directory that this script consumes. |
| **Down‑stream** | The hourly aggregation script (`$MNAASDailyTrafficDetails_Hourly_Aggr_SriptName`) consumes the populated Hive tables to produce aggregated reports used by billing/analytics pipelines. |
| **Shared Status File** | `$MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName` is also read/written by other daily load scripts to coordinate multi‑step jobs and avoid concurrent runs. |
| **Backup & Retention** | Backup directory (`$Daily_TrafficDetails_BackupDir`) may be accessed by archival or audit scripts. |
| **Ticketing/Alerting** | `email_and_SDP_ticket_triggering_step` integrates with the internal SDP ticketing system via email. |
| **Lock Mechanism** | `$MergeLockName` is likely created by a scheduler or a preceding script to signal that the merge step should run (e.g., only once per hour). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Early `exit` after `exec`** – the script contains `exec 2>>$log` **followed by `exit;`** before any function calls, which aborts the entire pipeline. | Complete data‑load failure, silent no‑op. | Remove the stray `exit` (or replace with `exec 2>>$log` only). Add a guard to ensure the log redirection persists without terminating. |
| **Process‑status flag corruption** – simultaneous runs could overwrite the flag file, causing inconsistent resumes. | Data loss or duplicate processing. | Enforce exclusive lock via `flock` around the whole script, or use atomic `mv` to replace the status file. |
| **Missing or malformed input files** – pattern check may allow unexpected files; duplicate removal may drop legitimate rows if they are identical across files. | Incorrect reporting, missing records. | Tighten filename regex, add checksum validation, and log duplicate counts for audit. |
| **HDFS/Hive command failures** – no retry logic; a transient network glitch aborts the job. | Partial load, downstream jobs fail. | Wrap Hadoop/Hive calls in a retry loop with exponential back‑off; capture exit codes more robustly. |
| **Lock file race** – if `$MergeLockName` is not removed correctly, merge step may be permanently skipped. | No merged file → downstream load fails. | Ensure lock removal is idempotent; consider using a timestamped lock and a cleanup cron. |
| **Resource exhaustion** – large daily files can exceed local disk or memory during duplicate removal (`awk`). | Script crash, OS OOM. | Process files in streaming mode (e.g., `sort -u` with temporary directory on larger storage) or split large files before de‑duplication. |
| **Email flood on repeated failures** – each failure may generate a ticket; if the flag is not reset, repeated alerts could be suppressed unintentionally. | Missed alerts or ticket spamming. | Add a cooldown timer and ensure the flag is cleared after ticket resolution. |

---

### 6. Execution & Debugging Guide  

1. **Typical Invocation** (via cron, e.g., `0 * * * * /app/hadoop_users/MNAAS/.../MNAAS_SimChange_tbl_Load.sh`)  
   - The script runs with `set -x` (debug trace) enabled, so each command is echoed to the log file.  

2. **Manual Run (debug mode)**  
   ```bash
   export MNAAS_DailyTrafficDetailsLoadAggrLogPath=/tmp/MNAAS_TrafficDetails.log
   export MNAAS_TrafficDetails_tbl_Load.properties=/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_TrafficDetails_tbl_Load.properties
   . $MNAAS_TrafficDetails_tbl_Load.properties   # load all required vars
   bash -x MNAAS_SimChange_tbl_Load.sh   # watch stdout & log
   ```  

3. **Checking Process Status**  
   ```bash
   grep MNAAS_Daily_ProcessFlag $MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName
   cat $MNAAS_DailyTrafficDetailsLoadAggrLogPath | tail -n 20
   ```  

4. **Common Debug Points**  
   - **File pattern mismatch** – verify `$Traffic_Details_extn` and the regex in `MNAAS_files_pre_validation`.  
   - **HDFS copy failure** – run the `hadoop fs -ls $MNAAS_Daily_Rawtablesload_TrafficDetails_PathName` manually.  
   - **Java JAR errors** – capture the stdout of the Java call by adding `>> $MNAAS_DailyTrafficDetailsLoadAggrLogPath 2>&1`.  
   - **Lock file** – ensure `$MergeLockName` exists when the script is expected to merge (usually at the top of the hour).  

5. **Log Rotation**  
   - The log file is appended to (`exec 2>>`). Set up a logrotate rule to compress and prune old logs (e.g., keep 30 days).  

---

### 7. External Configuration & Environment Variables  

| Variable (populated by the sourced properties file) | Meaning / Usage |
|---------------------------------------------------|-----------------|
| `MNAAS_DailyTrafficDetailsLoadAggrLogPath` | Path for the script’s error log (stderr redirected). |
| `MNAAS_TrafficDetails_tbl_Load.properties` | Property file sourced at the top; defines all paths, filenames, Hadoop/Hive parameters, Java class names, etc. |
| `MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName` | Central status/flag file used for checkpointing and PID tracking. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Root staging directory where raw CSVs land. |
| `Traffic_Details_extn` | Sub‑directory (or file extension) under the staging dir containing the traffic detail files. |
| `No_of_files_to_process` | Maximum number of files to pick per run (used with `head -$No_of_files_to_process`). |
| `MNAASInterFilePath_Daily_Traffic_Details` | Intermediate working directory for validated files. |
| `MNAASRejectedFilePath` | Destination for files that fail validation. |
| `EmptyFileDir` | Holds empty files removed during pre‑validation. |
| `Daily_TrafficDetails_BackupDir` | Backup location for a copy of the intermediate files. |
| `MNASS_Intermediatefiles_removedups_withdups_filepath` / `...withoutdups_filepath` | Temp dirs for duplicate‑removal steps. |
| `MNAAS_Daily_Rawtablesload_TrafficDetails_PathName` | HDFS target directory for the merged file. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components for the ETL JARs. |
| `Load_nonpart_table`, `Insert_Part_Daily_table`, `Insert_Part_Daily_Reject_table` | Fully‑qualified Java class names (or JAR entry points) invoked for Hive loads. |
| `dbname`, `traffic_details_daily_inter_tblname`, `traffic_details_daily_tblname`, `traffic_details_daily_reject_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `MNAASDailyTrafficDetails_Hourly_Aggr_SriptName`, `MNAASShellScriptPath` | Path and name of the hourly aggregation script invoked at the end. |
| `MergeLockName` | File whose presence signals that the merge step should be performed. |
| `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM` | Email addresses used for failure notifications. |
| `MNAAS_job_status`, `MNAAS_email_sdp_created` | Flags stored in the status file. |

*If any of the above variables are missing or point to non‑existent paths, the script will abort with an error logged to `$MNAAS_DailyTrafficDetailsLoadAggrLogPath`.*

---

### 8. Suggested Improvements (TODO)  

1. **Remove the stray `exit` after the log redirection** – it currently prevents the entire pipeline from executing. Replace with `exec 2>>"$MNAAS_DailyTrafficDetailsLoadAggrLogPath"` only.  
2. **Replace the many `sed -i` flag updates with an atomic write** (e.g., build the new status content in a temp file and `mv` it into place). This avoids race conditions when multiple scripts read/write the same status file.  

*Additional optional enhancements*: add retry logic for Hadoop/Hive commands, switch duplicate removal to a streaming `sort -u` to handle very large files, and externalize the lock‑file handling into a dedicated utility.  

--- 

*End of documentation.*