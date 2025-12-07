**File:** `move-mediation-scripts/bin/MNAAS_TrafficDetails_tbl_Load.sh`  

---

## 1. One‑paragraph production summary
This script orchestrates the end‑to‑end daily “Traffic Details” data‑move for the MNAAS mediation platform. It validates incoming CSV files, backs them up, strips headers, prefixes each record with its source filename, removes duplicate rows, merges the cleaned files, copies the merged payload to HDFS, and invokes a series of Java/Hive/Impala jobs that load the data into a temporary Hive table, populate the final raw and reject tables, generate partition metadata, and finally trigger the hourly aggregation job. Process state is persisted in a status file so that a partially‑completed run can be resumed without re‑processing already‑finished steps.

---

## 2. Key functions & responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Sets flag = 1, clears intermediate dir, selects the first *N* files from the staging area, validates filename pattern and that the file is readable text; moves good files (and accompanying `.sem` files) to the intermediate “Traffic Details” dir, rejects malformed files. |
| **MNAAS_cp_files_to_backup** | Flag = 2. Copies both CSV and `.sem` files from the intermediate dir to a daily backup directory. |
| **MNAAS_rm_header_from_files** | Flag = 3. Removes the first line (header) from every CSV in the intermediate dir. |
| **MNAAS_append_filename_to_start_of_records** | Flag = 4. Prefixes each record with `<filename>;` to retain source‑file provenance. |
| **MNAAS_move_files_to_another_temp_dir** | Flag = 5. Copies the prefixed CSVs to a second temporary workspace (`removedups_withdups`). |
| **MNAAS_remove_duplicates_in_the_files** | Flag = 6. De‑duplicates each file (awk `!a[$0]++`) and writes the unique rows to `removedups_withoutdups`. |
| **MNAAS_move_nodups_files_to_inter_dir** | Flag = 7. Moves the de‑duplicated CSVs back to the main intermediate dir and also to a “merge” staging area. |
| **MNAAS_mergefile_to_separate_dir** | Flag = 8. If a lock file (`$MergeLockName`) exists (hourly run), concatenates all CSVs in the merge staging area into a single file `Mergeinput<date>`; otherwise skips and exits successfully. |
| **MNAAS_load_files_into_temp_table** | Flag = 9. Copies the merged file to HDFS, then runs the Java class `$Load_nonpart_table` to bulk‑load into a temporary Hive table (`traffic_details_daily_inter_tblname`). |
| **MNAAS_insert_into_raw_table** | Flag = 10. Executes Java `$Insert_Part_Daily_table` to move data from the temp table to the final raw table, cleans up intermediate files, and refreshes the Impala table. |
| **MNAAS_insert_into_reject_table** | Flag = 11. Executes Java `$Insert_Part_Daily_Reject_table` to populate the reject table for rows that failed validation in the previous step. |
| **MNAAS_save_partitions** | Flag = 12. Runs a Hive/Beeline query that extracts distinct partition dates from the raw table and writes them to a file; the file is later used by downstream processes. |
| **MNAAS_insert_into_api_table** | Flag = 13. (Currently commented out – separate job handles this.) |
| **MNAAS_hourly_aggr** | Flag = 14. Calls the external hourly aggregation script (`$MNAASDailyTrafficDetails_Hourly_Aggr_SriptName`) with the current hour/minute. |
| **terminateCron_successful_completion** | Resets status flags to *0* (Success) and logs completion. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers SDP ticket/email via `email_and_SDP_ticket_triggering_step`, and exits with error. |
| **email_and_SDP_ticket_triggering_step** | Sends a templated email to the support mailbox and marks the ticket‑creation flag to avoid duplicate alerts. |

---

## 3. Inputs, outputs & side‑effects  

| Category | Details |
|----------|---------|
| **Input files** | • CSV files in `$MNAASMainStagingDirDaily_afr_seq_check/$Traffic_Details_extn` (first *N* defined by `$No_of_files_to_process`). <br>• Corresponding `.sem` files in `$MNAASMainStagingDirDaily_v2`. |
| **Configuration** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_TrafficDetails_tbl_Load.properties`. This file defines all path variables, Hadoop/Hive/Impala connection strings, Java class names, log locations, lock file name, etc. |
| **Environment variables** | Implicitly used: `$PATH`, `$CLASSPATHVAR`, `$Generic_Jar_Names`, `$MNAAS_Main_JarPath`, `$HIVE_HOST`, `$IMPALAD_HOST`, `$HIVE_JDBC_PORT`, `$IMPALAD_JDBC_PORT`, `$SDP_ticket_*`, `$MOVE_DEV_TEAM`. |
| **Outputs** | • Processed CSVs stored in `$Daily_TrafficDetails_BackupDir`. <br>• Merged file in `$MNAASInterFilePath_Daily_Traffic_Details_Merge`. <br>• HDFS files under `$MNAAS_Daily_Rawtablesload_TrafficDetails_PathName`. <br>• Hive/Impala tables: `traffic_details_daily_inter_raw_daily`, `traffic_details_daily_raw`, `traffic_details_daily_reject`. <br>• Partition list file `$MNAAS_Traffic_Partitions_Hourly_FileName`. <br>• Log file `$MNAAS_DailyTrafficDetailsLoadAggrLogPath`. |
| **Side‑effects** | • Updates the *process status* file (`$MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName`) with flags, PID, timestamps, and job status. <br>• May send email/SLA ticket on failure. <br>• Removes/archives empty or header‑only files. |
| **Assumptions** | • The staging directories exist and are writable by the script user. <br>• Hadoop, Hive, Impala services are reachable and the user has required permissions. <br>• Java JARs referenced contain the expected classes (`Load_nonpart_table`, `Insert_Part_Daily_table`, etc.). <br>• The lock file `$MergeLockName` is created by an upstream scheduler to indicate an hourly merge window. |

---

## 4. Interaction with other scripts / components  

| Connected component | How this script interacts |
|---------------------|---------------------------|
| **MNAAS_TrafficDetails_hourly_tbl_aggr.sh** | Invoked indirectly via `MNAAS_hourly_aggr` which calls `$MNAASDailyTrafficDetails_Hourly_Aggr_SriptName`. |
| **MNAAS_TrafficDetails_daily_tbl_aggr.sh** | Not called directly; the daily raw table populated here is later aggregated by that script. |
| **MNAAS_TrafficDetails_mergefile_triggerScript.sh** | Creates `$MergeLockName` that gates the merge step in this script. |
| **MNAAS_Tolling_tbl_Load.sh** (and driver) | Parallel pipeline that loads a different CDR domain; both share the same status‑file convention and may run concurrently. |
| **Java JARs** (`Load_nonpart_table`, `Insert_Part_Daily_table`, `Insert_Part_Daily_Reject_table`) | Executed via `java -cp …` to perform bulk loads into Hive/Impala. |
| **Hadoop/HDFS** | Uses `hadoop fs -rm`, `hadoop fs -copyFromLocal`, `hdfs dfs -cat` to stage data. |
| **Hive/Impala** | Runs `beeline` and `impala-shell` commands to refresh tables and execute partition queries. |
| **SDP ticketing/email** | On failure, `email_and_SDP_ticket_triggering_step` sends an email to `Cloudera.Support@tatacommunications.com` and copies to the MOVE dev team. |

---

## 5. Operational risks & mitigations  

| Risk | Mitigation |
|------|------------|
| **Incorrect filename pattern** – valid files silently rejected. | Validate pattern definitions in the script against source system naming conventions; add a configurable regex in the properties file. |
| **Duplicate removal may drop legitimate duplicate rows** (e.g., same call recorded twice). | Ensure deduplication key is appropriate; consider adding a checksum column before de‑duplication. |
| **Concurrent runs** – two instances could corrupt the status file. | The script checks the PID stored in the status file; enforce a lock file (`$MergeLockName`) at the very start of the script as an extra safeguard. |
| **HDFS space exhaustion** when copying large merged files. | Monitor HDFS usage; add a pre‑flight `hdfs dfs -du` check and abort with a clear log message if free space < threshold. |
| **Java job failure** (class not found, OOM). | Capture Java exit codes (already done) and forward the stack trace to the log; consider adding JVM memory options via properties. |
| **Missing/incorrect properties** – script aborts early. | Validate required variables after sourcing the properties file; exit with a descriptive error if any are empty. |
| **Lock file never cleared** → merge step never runs. | Ensure the upstream process that creates `$MergeLockName` also removes it on success/failure; add a timeout cleanup in this script. |
| **Email/SDP flood** if the same failure repeats. | The status file flag `MNAAS_email_sdp_created` prevents duplicate tickets; verify that the flag is reset on a successful run. |

---

## 6. Running / debugging the script  

1. **Typical invocation (cron):**  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_TrafficDetails_tbl_Load.sh
   ```
   No arguments → `reprocess=false`, dates set to a sentinel value.

2. **Force a re‑process of a specific date range:**  
   ```bash
   .../MNAAS_TrafficDetails_tbl_Load.sh -r true -s 20240101 -e 20240101
   ```
   The script validates the `YYYYMMDD` format before proceeding.

3. **Check current status flag:**  
   ```bash
   grep MNAAS_Daily_ProcessStatusFlag $MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName
   ```

4. **Enable verbose debugging:**  
   The script already runs with `set -x` (trace). To capture the trace in a separate file:  
   ```bash
   bash -x MNAAS_TrafficDetails_tbl_Load.sh > /tmp/debug.out 2>&1
   ```

5. **Manual step‑by‑step execution:**  
   Each function can be called individually from a shell after sourcing the properties file, e.g.:  
   ```bash
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_TrafficDetails_tbl_Load.properties
   MNAAS_files_pre_validation
   MNAAS_cp_files_to_backup
   # …etc.
   ```

6. **Log inspection:**  
   All operational messages are appended to `$MNAAS_DailyTrafficDetailsLoadAggrLogPath`. Search for “failed” or “ERROR” to locate problems.

---

## 7. External configuration / files referenced  

| File / Variable | Purpose |
|-----------------|---------|
| **MNAAS_TrafficDetails_tbl_Load.properties** | Centralizes all directory paths, Hadoop/Hive/Impala connection strings, Java class names, lock file name, log file locations, and other tunable parameters (`$No_of_files_to_process`, `$Traffic_Details_extn`, `$SEM_EXTN`, etc.). |
| **Process status file** (`$MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName`) | Persists flags, PID, timestamps, job status, and email‑ticket flag across runs. |
| **Merge lock file** (`$MergeLockName`) | Indicates that the hourly merge window is open; created by an upstream scheduler. |
| **Java JARs** (`$Generic_Jar_Names`, `$MNAAS_Main_JarPath`) | Contain the compiled classes invoked for loading and inserting data. |
| **Email / SDP variables** (`$SDP_ticket_from_email`, `$SDP_ticket_to_email`, `$MOVE_DEV_TEAM`) | Used by the failure‑notification routine. |
| **Backup directories** (`$Daily_TrafficDetails_BackupDir`, `$EmptyFileDir`, `$MNAASRejectedFilePath`) | Hold copies of good, empty, and rejected files respectively. |

---

## 8. Suggested improvements (TODO)

1. **Parameterise the filename‑validation regex** in the properties file so that changes to upstream naming conventions do not require script edits.  
2. **Add a pre‑run sanity check** that verifies all required environment variables and directories exist (exit early with a clear error) – this will make troubleshooting faster and prevent silent failures later in the pipeline.  

--- 

*End of documentation.*