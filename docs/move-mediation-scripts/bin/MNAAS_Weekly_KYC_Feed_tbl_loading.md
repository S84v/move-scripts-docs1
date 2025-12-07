**MNAAS_Weekly_KYC_Feed_tbl_loading.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

This Bash script orchestrates the weekly ingestion of KYC (Know‑Your‑Customer) feed files into the MNAAS data‑warehouse. It validates incoming CSV files, moves malformed or empty files to quarantine, backs up validated files, strips headers, de‑duplicates records, loads the cleaned data into a temporary Hive/Impala table via Hadoop FS and a Java loader, and finally merges the data into the production KYC snapshot table. The script maintains a process‑status file for restartability, logs every step, and raises an SDP ticket with email notification on failure.

---

### 2. Core Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | • Set status flag = 1.<br>• Clean staging directories.<br>• List incoming files, validate naming (`*_WEEKLY_*.csv`) and that they are readable text.<br>• Move good files to `Weekly_KYC_Feed_Details`; bad files to `Rejected`.<br>• Separate empty files and header‑only files to `EmptyFileDir`.<br>• Abort if no files remain. |
| **MNAAS_cp_files_to_backup** | • Set status flag = 2.<br>• Copy all validated files to the weekly backup directory. |
| **MNAAS_rm_header_from_files** | • Set status flag = 3.<br>• Prepend the source filename to each record (`filename,record`).<br>• Remove the CSV header line (`iccid`) and append the result to a daily‑incremental “dups” file. |
| **MNAAS_remove_duplicates_in_the_files** | • Set status flag = 4.<br>• For each incremental file, count rows before/after de‑duplication, write unique rows to a one‑time snapshot file, and delete the processed incremental file. |
| **MNAAS_load_files_into_temp_table** | • Set status flag = 5.<br>• Ensure no concurrent loader (`ps` check).<br>• Remove any previous HDFS raw‑load path, copy cleaned files to HDFS.<br>• Invoke Java class `$Load_nonpart_table` to bulk‑load into Hive temp table.<br>• Refresh Impala metadata. |
| **MNAAS_insert_into_kyc_snapshot_table** | • Set status flag = 6.<br>• Ensure no concurrent inserter.<br>• Run Java class `$Insert_Part_Daily_table` to merge temp data into the production KYC snapshot table.<br>• Clean up staging files and refresh Impala. |
| **terminateCron_successful_completion** | • Reset status flag = 0, mark job as *Success*, write run‑time, log completion, exit 0. |
| **terminateCron_Unsuccessful_completion** | • Log failure, invoke `email_and_SDP_ticket_triggering_step`, exit 1. |
| **email_and_SDP_ticket_triggering_step** | • Mark job status *Failure* in the status file.<br>• If no ticket has been raised, send an email (via `mailx`) to the SDP ticketing address and flag the ticket as created. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input files** | • Weekly KYC feed files in `$MNAASMainStagingDirDaily_afr_seq_check/$Weekly_KYC_Feed_Weekly_Usage_ext` (max `$No_of_files_to_process`).<br>• Corresponding SHA256 checksum files (`*.sha256`). |
| **Configuration** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed_tbl_Load.properties`. This file defines all path variables, Hadoop/Hive/Impala connection strings, Java class/jar names, process‑status file, log file, backup dirs, etc. |
| **Outputs** | • Cleaned, de‑duplicated CSV files in `$Weekly_KYC_BackupDir` (backup) and `$Weekly_Increment_KYC_Feed_WithoutDups` (temp).<br>• HDFS raw load directory `$MNAAS_Weekly_Rawtablesload_KYC_PathName` populated with cleaned files.<br>• Hive/Impala tables `kyc_weekly_inter_tblname` and `kyc_iccid_wise_country_hist_tblname` updated.<br>• Process‑status file updated throughout the run.<br>• Log file `$MNAAS_Weekly_KYC_Feed_Load_LogPath`. |
| **Side effects** | • Files moved to `Rejected`, `EmptyFileDir`, or backup directories.<br>• HDFS delete (`hadoop fs -rm`) of previous raw‑load path.<br>• Impala metadata refresh commands.<br>• Potential SDP ticket creation via email. |
| **Assumptions** | • All required environment variables are defined in the sourced properties file.<br>• Hadoop, Hive, Impala services are reachable and the user has required permissions (HDFS write, Hive/Impala access).<br>• Java runtime and required JARs are present on the classpath.<br>• No other instance of the loader/inserter is running (checked via `ps`). |

---

### 4. Interaction with Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **MNAAS_Daily_KYC_Feed_tbl_Load.properties** | Central configuration; shared with daily KYC load scripts. |
| **Other “MNAAS_*” scripts** (e.g., `MNAAS_Traffic_tbl_*`, `MNAAS_Usage_Trend_Aggr.sh`) | Use the same process‑status file pattern and logging conventions; may run on the same schedule but on different data domains. |
| **Java loader JARs** (`$Load_nonpart_table`, `$Insert_Part_Daily_table`) | Executed from this script to perform bulk Hive loads and merges. |
| **Hadoop/HDFS** | Source/target for raw CSV files (`copyFromLocal`, `rm`). |
| **Hive / Impala** | Destination tables; refreshed via `impala-shell`. |
| **SDP ticketing system** | Email‑based ticket creation on failure (addresses defined in properties). |
| **Cron scheduler** | The script is intended to be invoked by a nightly/weekly cron job; the PID guard prevents overlapping runs. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale PID / orphaned lock** – script may think a previous run is still active. | Add a timeout check on the stored PID (e.g., verify process start time) and optionally clean the lock after a configurable max runtime. |
| **Missing or malformed properties file** – all path variables become empty, causing file loss. | Validate required variables at start of script; abort with clear error if any are undefined. |
| **Hadoop/Hive/Impala connectivity failure** – data not loaded, but files may already be moved. | Implement retry logic for `hadoop fs` and `impala-shell` commands; on repeated failure, move files back to a “reprocess” directory. |
| **Duplicate removal logic (`awk '!a[$0]++'`) may be memory‑intensive for large files.** | Switch to a streaming deduplication tool (e.g., Hadoop MapReduce or Spark) for very large datasets. |
| **File naming validation regex is too strict** – legitimate files could be rejected. | Document the expected naming convention and make the regex configurable. |
| **Hard‑coded `chmod -R 777`** – security exposure. | Replace with least‑privilege permissions; use a dedicated service account. |
| **Email/SDP ticket flood on repeated failures** – same failure may generate many tickets. | Add a back‑off counter in the status file to limit ticket creation frequency. |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed_tbl_Load.properties` is present and contains all required variables.  
   - Verify Hadoop, Hive, Impala, Java, and `mailx` are in the `$PATH`.  

2. **Execution**  
   ```bash
   # As the MNAAS service user
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Weekly_KYC_Feed_tbl_loading.sh
   ```
   The script is normally scheduled via cron (e.g., `0 2 * * 1` for weekly run).  

3. **Debugging**  
   - The script starts with `set -x`; all commands are echoed to the log file `$MNAAS_Weekly_KYC_Feed_Load_LogPath`.  
   - Tail the log while the job runs: `tail -f $MNAAS_Weekly_KYC_Feed_Load_LogPath`.  
   - To force a fresh run, reset the status flag in the process‑status file to `0` or `1`.  
   - If a step fails, the script logs the error and exits with status 1; the status file will contain `MNAAS_job_status=Failure`.  

4. **Manual Step‑by‑Step** (useful for testing)  
   ```bash
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed_tbl_Load.properties
   MNAAS_files_pre_validation
   MNAAS_cp_files_to_backup
   MNAAS_rm_header_from_files
   MNAAS_remove_duplicates_in_the_files
   MNAAS_load_files_into_temp_table   # may require Hadoop/Java access
   MNAAS_insert_into_kyc_snapshot_table
   terminateCron_successful_completion
   ```

---

### 7. External Configuration & Environment Variables

| Variable (defined in properties) | Role |
|----------------------------------|------|
| `MNAAS_Weekly_KYC_Feed_Load_ProcessStatusFileName` | Path to the status file that stores flags, PID, timestamps, etc. |
| `MNAAS_Weekly_KYC_Feed_Load_LogPath` | Central log file for this run. |
| `MNAASMainStagingDirDaily_afr_seq_check` / `Weekly_KYC_Feed_Weekly_Usage_ext` | Source directory and file pattern for incoming weekly KYC CSVs. |
| `MNAASInterFilePath_Weekly_KYC_Feed_Details` | Working directory for validated files. |
| `MNAASRejectedFilePath`, `EmptyFileDir` | Quarantine directories for bad/empty files. |
| `Weekly_KYC_BackupDir` | Backup location for validated files. |
| `Weekly_Increment_KYC_Feed_Dups`, `KYC_OneTimeSnapshotWeekly` | Intermediate files for de‑duplication. |
| `MNAAS_Weekly_Rawtablesload_KYC_PathName` | HDFS target directory for raw load. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `Load_nonpart_table`, `Insert_Part_Daily_table` | Java classpath and main class names used for bulk load and merge. |
| `dbname`, `kyc_weekly_inter_tblname`, `kyc_iccid_wise_country_hist_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `SDP_ticket_to_email`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email addresses for failure notifications. |
| `No_of_files_to_process` | Upper limit on files processed per run. |
| `Dname_MNAAS_Load_Weekly_KYC_tbl_temp`, `Dname_MNAAS_Insert_Weekly_KYC_tbl` | Process names used for PID checks. |

If any of these variables are missing or empty, the script will likely fail early; a sanity‑check block at the top of the script is recommended.

---

### 8. Suggested Improvements (TODO)

1. **Add a pre‑flight validation block** that checks all required variables from the properties file, verifies directory existence and write permissions, and aborts with a clear error if any check fails.  
2. **Replace the ad‑hoc PID lock with a robust lockfile mechanism** (e.g., `flock`) that includes a timeout and automatic cleanup, reducing the risk of stale locks and simplifying concurrency control.  

---