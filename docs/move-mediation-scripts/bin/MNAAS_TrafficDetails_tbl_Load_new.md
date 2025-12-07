**File:** `move-mediation-scripts\bin\MNAAS_TrafficDetails_tbl_Load_new.sh`  

---

## 1. High‑Level Summary
This Bash script is the orchestrator for the **daily “TrafficDetails” load pipeline** used by the MNAAS mediation platform. It ingests a configurable set of CSV‑style traffic‑detail files from a staging directory, validates format, removes headers, tags each record with its source filename, de‑duplicates, merges into a single file, and loads the data into a Hive/Impala temporary table. Subsequent Java jobs move the data into the final raw and reject tables, refresh partitions, push a subset to an API‑exposure table (and onward to Oracle), and finally trigger an hourly aggregation job. The script maintains a persistent “process‑status” file to allow safe restart from any step and sends an email/SDP ticket on failure.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Sets flag = 1, clears previous intermediate files, selects the first *N* files (configurable) from the staging area, validates filename pattern and that the file is readable text. Moves good files to `$MNAASInterFilePath_Daily_Traffic_Details`, bad files to `$MNAASRejectedFilePath`. Filters out empty files and header‑only files. |
| **MNAAS_cp_files_to_backup** | Flag = 2. Copies the validated intermediate files to a daily backup directory. |
| **MNAAS_rm_header_from_files** | Flag = 3. Strips the first line (header) from each intermediate file. |
| **MNAAS_append_filename_to_start_of_records** | Flag = 4. Prefixes each record with the source filename (`filename;record`). |
| **MNAAS_move_files_to_another_temp_dir** | Flag = 5. Copies the processed files to a second temp directory (`$MNASS_Intermediatefiles_removedups_withdups_filepath`). |
| **MNAAS_remove_duplicates_in_the_files** | Flag = 6. De‑duplicates each file using `awk '!a[$0]++'`. |
| **MNAAS_move_nodups_files_to_inter_dir** | Flag = 7. Moves de‑duplicated files back to the primary intermediate dir and also to a merge‑ready dir. |
| **MNAAS_mergefile_to_separate_dir** | Flag = 8. If a lock file (`$MergeLockName`) exists, concatenates all merge‑ready files into a single “Mergeinput” file, then cleans up. If the lock is absent, skips merge and exits successfully. |
| **MNAAS_load_files_into_temp_table** | Flag = 9. Copies the merged file to HDFS, then runs a Java class (`$Load_nonpart_table`) to bulk‑load into a Hive temporary table. Refreshes the Impala view. |
| **MNAAS_insert_into_raw_table** | Flag = 10. Executes a Java class (`$Insert_Part_Daily_table`) to insert from the temp table into the final raw table, then cleans intermediate directories and refreshes Impala. |
| **MNAAS_insert_into_reject_table** | Flag = 11. Executes a Java class (`$Insert_Part_Daily_Reject_table`) to move rejected rows to a reject table and refreshes Impala. |
| **MNAAS_save_partitions** | Flag = 12. Queries Hive for distinct `calldate` partitions and writes them to `$MNAAS_Traffic_Partitions_Hourly_FileName`. |
| **MNAAS_insert_into_api_table** | Flag = 13. Truncates and loads an API‑exposure table via Impala, then runs an Oracle‑loading shell script (`$api_med_data_oracle_loadingSriptName`). |
| **MNAAS_hourly_aggr** | Flag = 14. Calls a separate hourly‑aggregation script (`$MNAASDailyTrafficDetails_Hourly_Aggr_SriptName`) passing the current hour/minute. |
| **terminateCron_successful_completion** | Resets status flags to *0* (Success) and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends an email (via `mailx`) to the support team and marks the SDP‑ticket‑created flag to avoid duplicate alerts. |

The **main block** reads the persisted flag from `$MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName` and, based on its value, re‑executes the pipeline from the appropriate step. It also prevents concurrent runs by checking the PID stored in the same status file.

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Configuration (properties file)** | `MNAAS_TrafficDetails_tbl_Load.properties` – defines all `$MNAAS*` variables: staging dirs, file extensions, number of files to process, backup paths, Hadoop/Impala/Hive connection info, Java class names & JAR locations, email/SDP parameters, lock file name, etc. |
| **Environment variables** | Implicitly used: `PATH`, `JAVA_HOME`, `HADOOP_CONF_DIR`, `HIVE_CONF_DIR`. The script itself does not read them directly but the called Java/JARs and Hive/Impala commands do. |
| **External services** | - **HDFS** (copyFromLocal, rm) <br> - **Hive** (DDL/queries) <br> - **Impala** (shell refresh commands) <br> - **Java** jobs (`Load_nonpart_table`, `Insert_Part_Daily_table`, `Insert_Part_Daily_Reject_table`) <br> - **Mailx** (SDP ticket) <br> - **Oracle** (via `$api_med_data_oracle_loadingSriptName`) |
| **File system inputs** | - Raw traffic‑detail files in `$MNAASMainStagingDirDaily_afr_seq_check/$Traffic_Details_extn` <br> - Status/process flag file `$MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName` |
| **File system outputs** | - Validated/processed CSV files in several intermediate dirs (`$MNAASInterFilePath_Daily_Traffic_Details`, `$MNASS_Intermediatefiles_*`, `$MNAASInterFilePath_Daily_Traffic_Details_Merge*`) <br> - Backup copies in `$Daily_TrafficDetails_BackupDir` <br> - Merged file on HDFS (`$MNAAS_Daily_Rawtablesload_TrafficDetails_PathName`) <br> - Log file `$MNAAS_DailyTrafficDetailsLoadAggrLogPath` <br> - Partition list file `$MNAAS_Traffic_Partitions_Hourly_FileName` |
| **Side‑effects** | - Updates status flag file (process flag, PID, job status, timestamps). <br> - Sends email on failure. <br> - May create/delete lock file `$MergeLockName`. |
| **Assumptions** | - The properties file exists and defines all referenced variables. <br> - Required JARs and Java classes are present and compatible with the Hadoop/Cloudera version. <br> - Hive/Impala services are reachable. <br> - Sufficient disk space for intermediate files and HDFS staging area. <br> - The naming convention for incoming files matches the regex `^[a-zA-Z]+_[0-9]+_[a-zA-Z]+_[0-9]+_[0-9]+\.csv$`. |

---

## 4. Interaction with Other Scripts / Components  

| Component | Connection Point |
|-----------|------------------|
| **`MNAAS_TrafficDetails_tbl_Load.properties`** | Sourced at the top; provides all path & job parameters. |
| **`$MNAASDailyTrafficDetails_Hourly_Aggr_SriptName`** | Invoked by `MNAAS_hourly_aggr` with current hour/minute. |
| **`$api_med_data_oracle_loadingSriptName`** | Called after API table load to push data into Oracle. |
| **Java JARs** (`$Load_nonpart_table`, `$Insert_Part_Daily_table`, `$Insert_Part_Daily_Reject_table`) | Executed via `java -cp $CLASSPATHVAR ...`. |
| **Lock file `$MergeLockName`** | Controls whether the merge step runs (only when lock present). |
| **SDP ticketing / email** | `email_and_SDP_ticket_triggering_step` uses `mailx` with addresses defined in the properties file (`$SDP_ticket_to_email`, `$SDP_ticket_from_email`, `$MOVE_DEV_TEAM`). |
| **Other daily load scripts** | The status flag file is shared across runs; if a previous run failed at step X, the next execution resumes from step X+1. |
| **Cron scheduler** | The script is intended to be launched by a cron job (the PID check prevents overlapping runs). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect file naming pattern** – files silently moved to reject dir. | Data loss / delayed processing. | Add a configurable regex in the properties file; log the rejected filenames with reason; optionally alert if reject count > threshold. |
| **Duplicate removal using `awk` may be memory‑intensive for large files**. | Job hangs or OOM. | Switch to Hadoop streaming or Spark for de‑duplication; or process files in chunks. |
| **Lock file misuse** – missing lock prevents merge, causing downstream steps to run on empty data. | Incomplete loads. | Ensure lock creation is part of a preceding step; monitor lock presence; add fallback to create lock if absent after a certain time window. |
| **Concurrent runs** – PID check may be bypassed if the status file is corrupted. | Data corruption. | Store PID in a separate lock file with `flock`; also check process existence via `ps -p $PID`. |
| **Hard‑coded `sed` edits to status file** – race conditions if multiple scripts edit simultaneously. | Inconsistent status. | Centralize status handling in a single process or use atomic file writes (`mv` temp file). |
| **External service failures (Hive/Impala/Java jobs)** – script only checks exit code of the immediate command. | Partial data load, orphaned temp files. | Implement retry logic with exponential back‑off; capture and archive error output for post‑mortem. |
| **Mailx failure** – SDP ticket not created. | No alert to support. | Verify mailx return code; fallback to a secondary notification channel (e.g., Slack webhook). |
| **Hard‑coded paths** – may break after migration or OS change. | Script stops. | Move all paths to the properties file; validate existence at start. |

---

## 6. Running / Debugging the Script  

1. **Typical invocation (via cron)**  
   ```bash
   05 02 * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_TrafficDetails_tbl_Load_new.sh >> /dev/null 2>&1
   ```
2. **Manual run (for testing)**  
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_TrafficDetails_tbl_Load_new.sh   # -x already set, logs to $MNAAS_DailyTrafficDetailsLoadAggrLogPath
   ```
3. **Debug steps**  
   - Verify the properties file loads (`echo $MNAASMainStagingDirDaily_afr_seq_check`).  
   - Tail the log file while the script runs: `tail -f $MNAAS_DailyTrafficDetailsLoadAggrLogPath`.  
   - If a step fails, the log will show the function name and the exit status; re‑run that function manually (e.g., `MNAAS_load_files_into_temp_table`) after fixing the underlying issue.  
   - Check the status flag file to see the last successful step: `cat $MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName`.  
   - Use `ps -ef | grep MNAAS_Script_Process_Id` to ensure no stray processes remain.  

4. **Common troubleshooting**  
   - **Missing JAR** → `java -cp` error → verify `$CLASSPATHVAR` includes all required JARs.  
   - **HDFS permission** → `hadoop fs -copyFromLocal` fails → ensure the user running the script has write permission on `$MNAAS_Daily_Rawtablesload_TrafficDetails_PathName`.  
   - **Impala refresh** → `impala-shell` returns non‑zero → check Impala service health.  

---

## 7. External Config / Environment Dependencies  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_DailyTrafficDetailsLoadAggrLogPath` | Central log file for this run. |
| `MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName` | Persistent status/flag file. |
| `MNAASMainStagingDirDaily_afr_seq_check` / `Traffic_Details_extn` | Source directory & file extension for incoming traffic files. |
| `No_of_files_to_process` | Max number of files to handle per run. |
| `MNAASInterFilePath_Daily_Traffic_Details` | Primary intermediate directory. |
| `MNAASRejectedFilePath` / `EmptyFileDir` | Rejection and empty‑file handling dirs. |
| `Daily_TrafficDetails_BackupDir` | Backup location for raw files. |
| `MNASS_Intermediatefiles_removedups_*` | Temp dirs used for duplicate removal. |
| `MergeLockName` | Presence triggers merge step. |
| `MNAAS_Daily_Rawtablesload_TrafficDetails_PathName` | HDFS target directory for the merged file. |
| `Load_nonpart_table`, `Insert_Part_Daily_table`, `Insert_Part_Daily_Reject_table` | Fully‑qualified Java class names invoked via `java`. |
| `MNAAS_Main_JarPath`, `CLASSPATHVAR` | JAR locations for Java jobs. |
| `dbname`, `traffic_details_daily_inter_tblname`, `traffic_details_daily_tblname`, `traffic_details_daily_reject_tblname` | Hive database & table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints. |
| `api_med_data_oracle_loadingSriptName` | Shell script that loads API data into Oracle. |
| `MNAASDailyTrafficDetails_Hourly_Aggr_SriptName` | Hourly aggregation script. |
| `SDP_ticket_to_email`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email routing for failure alerts. |
| `MNAAS_Script_Process_Id` | PID stored for concurrency guard. |

All of the above must be defined; otherwise the script will abort with a “command not found” or “file not found” error.

---

## 8. Suggested Improvements (TODO)

1. **Refactor status handling** – Replace the series of `sed` edits on a flat file with a small SQLite or JSON state file written atomically. This eliminates race conditions and makes the flag values self‑describing.

2. **Modularize duplicate removal** – Move the de‑duplication step to a Spark job (or Hive `INSERT OVERWRITE … SELECT DISTINCT`) to handle large volumes efficiently and to leverage cluster resources instead of a single‑node `awk` process.

--- 

*End of documentation.*