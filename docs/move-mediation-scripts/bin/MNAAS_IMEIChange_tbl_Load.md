**File:** `move-mediation-scripts\bin\MNAAS_IMEIChange_tbl_Load.sh`  

---

## 1. High‑Level Summary
This script implements the end‑to‑end daily ingestion pipeline for the **IMEI‑Change** supplier feed. It discovers new CSV files in the staging area, validates naming and content, moves good files to an intermediate folder, backs them up, strips headers, prefixes each record with the source filename, loads the data into HDFS, runs two Java MapReduce/Impala jobs to populate a temporary Hive table and then merge into the final raw table, and finally updates a process‑status file. On any failure it logs the error, sends an email and raises an SDP ticket. The script is orchestrated by a cron job and is coordinated with other “MNAAS” feed‑load scripts through a shared status file.

---

## 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Scans the staging directory for the newest *N* files, validates filename pattern and that the file is a readable text file. Moves valid files to `$MNAASInterFilePath_IMEIChange`, rejects malformed files, logs actions. |
| **MNAAS_cp_files_to_backup** | Copies the intermediate files to a daily backup directory (`$Daily_IMEIChange_BackupDir`). |
| **MNAAS_rm_header_from_files** | Removes the first line (header) from each intermediate CSV. |
| **MNAAS_append_filename_to_start_of_records** | Prefixes every record with the source filename followed by a semicolon (`filename;record`). |
| **MNAAS_mv_files_to_hdfs** | Clears the target HDFS raw‑load path, then copies all intermediate files into `$MNAAS_Daily_Rawtablesload_IMEIChange_hdfsPathName`. |
| **MNAAS_load_files_into_temp_table** | Executes a Java job (`$Load_nonpart_table`) that reads the HDFS files and loads them into a temporary Hive table (`$IMEIChange_feed_inter_tblname`). Refreshes the Impala metadata. |
| **MNAAS_insert_into_raw_table** | Executes a second Java job (`$Insert_Part_Daily_table`) that merges the temporary table into the final raw table (`$IMEIChange_feed_tblname`). Refreshes Impala metadata. |
| **terminateCron_successful_completion** | Writes “Success” to the process‑status file, logs completion timestamps, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email and creates an SDP ticket (via `mailx`). Updates status flags to avoid duplicate alerts. |
| **Main script block** | Implements a lock‑file / PID check, reads the current process flag from the status file, and drives the appropriate subset of functions to resume from the last successful step. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input files** | CSV files matching `*IMEIChange*.csv` in `$MNAASMainStagingDirDaily_afr_seq_check`. Expected naming pattern: `<alpha>_<num>_<alpha>_<num>_<num>.csv`. |
| **Configuration** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_IMEIChange_tbl_Load.properties`. Contains all path, DB, Hadoop, Java‑jar, email, and flag variables referenced throughout the script. |
| **External services** | - HDFS (`hadoop fs` commands) <br> - Hive / Impala (via Java jobs and `impala-shell`) <br> - SMTP (mail, mailx) for alerts <br> - SDP ticketing system (email to `insdp@tatacommunications.com`) |
| **Outputs** | - Process‑status file (`$MNAAS_IMEIChange_tbl_Load_ProcessStatusFileName`) updated with flags, timestamps, PID, job status. <br> - Log file (`$MNAAS_IMEIChange_tbl_Load_log_path`). <br> - Backup copies of source CSVs (`$Daily_IMEIChange_BackupDir`). <br> - Populated Hive tables (`$IMEIChange_feed_inter_tblname`, `$IMEIChange_feed_tblname`). |
| **Side effects** | - Moves/rejects source files (to reject or empty‑file directories). <br> - Deletes HDFS target directory before copy. <br> - May generate duplicate email/SDP tickets if flag handling fails. |
| **Assumptions** | - The properties file defines all required variables and points to existing directories with proper permissions. <br> - Java JARs (`$Load_nonpart_table`, `$Insert_Part_Daily_table`) are compatible with the current Hadoop/Hive versions. <br> - Only one instance runs at a time (PID lock). <br> - Impala service is reachable (`$IMPALAD_HOST`). |

---

## 4. Integration with Other Scripts & Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **MNAAS_IMEIChange_seq_check.sh** | Generates the staging directory `$MNAASMainStagingDirDaily_afr_seq_check` and ensures sequential file arrival. |
| **MNAAS_HDFSSpace_Checks.sh** | Typically runs before this script to verify sufficient HDFS space for `$MNAAS_Daily_Rawtablesload_IMEIChange_hdfsPathName`. |
| **MNAAS_Daily_Validations_Checks.sh** | May be invoked upstream to perform additional data‑quality checks on the same feed. |
| **Other feed‑load scripts (e.g., MNAAS_GBS_Load.sh, MNAAS_Tolling_tbl_Aggr.sh)** | Share the same process‑status file naming convention and logging infrastructure; they run on the same schedule but on different feed directories. |
| **Cron scheduler** | The script is scheduled (typically nightly) via a crontab entry that sets required environment variables (e.g., `JAVA_HOME`, `HADOOP_CONF_DIR`). |
| **Monitoring / Alerting** | Logs are tail‑ed by Ops tools; the status file is polled to surface “Success/Failure” in dashboards. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID lock** – script crashes leaving PID in status file, preventing next run. | Daily ingestion stops. | Add a timeout check (e.g., if PID older than X hours, clear lock) and/or use a lock file with `flock`. |
| **Incorrect file naming or corrupted CSV** – files moved to reject folder, but downstream jobs may still run with zero input. | Data loss / missing records. | Implement a pre‑run alert when reject count > 0; verify naming regex with telecom standards. |
| **HDFS path deletion failure** – `hadoop fs -rm` fails, leaving old files that cause duplicate loads. | Duplicate rows, storage bloat. | Check exit status of `hadoop fs -rm` and abort with clear error if non‑zero. |
| **Java job failure (out‑of‑memory, schema mismatch)** – temporary or raw table not populated. | Incomplete data, downstream analytics break. | Capture Java job logs, set JVM heap limits, add schema validation step before load. |
| **Email/SDP ticket flood** – repeated failures may generate many tickets. | Alert fatigue. | Use the `MNAAS_email_sdp_created` flag (already present) and add a back‑off counter. |
| **Insufficient disk space in backup or intermediate directories** – `cp` or `mv` fails. | Data loss, script aborts. | Run `df` checks at start; integrate with `MNAAS_HDFSSpace_Checks.sh`. |
| **Permission changes on HDFS or local dirs** – script cannot read/write. | Job failure. | Ensure the script runs under a dedicated service account with proper ACLs; audit periodically. |

---

## 6. Running & Debugging the Script  

1. **Standard execution** (normally via cron):  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_IMEIChange_tbl_Load.properties
   /path/to/MNAAS_IMEIChange_tbl_Load.sh
   ```
   The script sources the properties file automatically.

2. **Manual run (debug mode)**:  
   ```bash
   set -x   # uncomment the line at top of script for bash tracing
   export MNAAS_IMEIChange_tbl_Load_SriptName=MNAAS_IMEIChange_tbl_Load.sh   # if not set in properties
   ./MNAAS_IMEIChange_tbl_Load.sh
   ```
   - Watch console for `+` trace lines.  
   - Tail the log file (`tail -f $MNAAS_IMEIChange_tbl_Load_log_path`).  

3. **Checking lock status**:  
   ```bash
   grep MNAAS_Script_Process_Id $MNAAS_IMEIChange_tbl_Load_ProcessStatusFileName
   ps -p <PID>
   ```

4. **Inspecting intermediate files**:  
   - Valid files: `$MNAASInterFilePath_IMEIChange`  
   - Rejected files: `$MNAASRejectedFilePath`  
   - Empty files moved to: `$EmptyFileDir`  

5. **Verifying Hive/Impala tables**:  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "SHOW TABLES LIKE '$IMEIChange_feed_tblname'; SELECT COUNT(*) FROM $IMEIChange_feed_tblname;"
   ```

6. **Java job logs**:  
   The Java commands write to `$MNAAS_IMEIChange_tbl_Load_log_path`; search for `ERROR` or stack traces.

---

## 7. External Configuration & Environment Variables  

| Variable (defined in properties) | Purpose |
|--------------------------------|---------|
| `MNAAS_IMEIChange_tbl_Load_log_path` | Path to the script’s log file. |
| `MNAAS_IMEIChange_tbl_Load_ProcessStatusFileName` | Central status/flag file used for checkpointing and PID lock. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Directory where supplier drops raw CSVs. |
| `IMEIChange_Details_extn` | Filename pattern (`*IMEIChange*.csv`). |
| `No_of_files_to_process` | Max number of files to pick per run. |
| `MNAASInterFilePath_IMEIChange` | Intermediate folder for validated files. |
| `MNAASRejectedFilePath` | Destination for malformed files. |
| `EmptyFileDir` | Holds empty CSVs after validation. |
| `Daily_IMEIChange_BackupDir` | Daily backup location for source files. |
| `MNAAS_Daily_Rawtablesload_IMEIChange_hdfsPathName` | HDFS target directory for raw load. |
| `Dname_MNAAS_Load_Daily_IMEIChange_tbl_temp` / `Dname_MNAAS_Load_Daily_IMEIChange_tbl` | Process names used to detect concurrent Java jobs. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath and JAR locations. |
| `Load_nonpart_table`, `Insert_Part_Daily_table` | Main class names for the two Java loaders. |
| `nameNode`, `dbname`, `IMEIChange_feed_inter_tblname`, `IMEIChange_feed_tblname` | Hive/Impala connection and table identifiers. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Service endpoints. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email notification parameters. |
| `MNAAS_IMEIChange_tbl_Load_SriptName` | Script name used in logs and alerts. |

If any of these variables are missing or point to non‑existent paths, the script will abort with an error logged in the log file.

---

## 8. Suggested TODO / Improvements  

1. **Add robust lock handling** – replace the manual PID check with `flock` and a timeout to automatically clear stale locks.  
2. **Centralize exit‑code handling** – create a generic `run_step` wrapper that logs start/end timestamps, captures exit status, and updates the status file, reducing repetitive code in each function.  

---