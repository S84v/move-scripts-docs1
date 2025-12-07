**File:** `move-mediation-scripts\bin\MNAAS_Daily_Recon.sh`

---

## 1. High‑Level Summary
`MNAAS_Daily_Recon.sh` is the orchestrator for the daily reconciliation workflow of the MNAAS (Move‑NL‑RA) reporting pipeline. It pulls daily Excel reconciliation files from a staging area, converts them to a CSV via a Java utility, loads the CSV into a temporary Hive table, merges it into the production partitioned table, archives the source files, and finally runs a Java‑based reconciliation report generator (including a previous‑month run on the 1st‑2nd of each month). The script maintains a persistent process‑status file to enable restartability and to prevent concurrent executions. All steps are logged, and failures trigger alert e‑mail and an SDP ticket.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_move_files_to_load** | Verify presence of HOL & SNG Excel files, move all recon files matching `$recon_file_format` into a `loading` sub‑directory, update status flag to **1**. |
| **MNAAS_export_files_to_csv** | Truncate the target CSV, invoke the Java JAR (`$MNAAS_Recon_JarPath`) to convert the Excel files to CSV, update status flag to **2**. |
| **MNAAS_load_temp_table** | Remove any stale CSV from HDFS, upload the new CSV, set permissions, truncate Hive temp table `mnaas.movenl_ra_recon_report_temp`, load CSV into it, refresh Impala metadata, update status flag to **3**. |
| **MNAAS_load_main_table** | Insert data from the temp table into the partitioned production table `mnaas.movenl_ra_recon_report` (partitioned by `year_month`), refresh Impala, update status flag to **4**. |
| **MNAAS_move_files_to_backup** | Archive processed Excel files from `loading` to `$MNAASReconBackupDir`, update status flag to **5**. |
| **MNAAS_perform_daily_recon** | Run the Java JAR to generate the daily reconciliation report (process name `$recon_process_name`). On the 1st‑2nd of each month, also generate a report for the previous month (handles year‑rollover). Updates status flag to **6**. |
| **send_alert_mail** | Send a plain‑text alert e‑mail using `mailx` to `$recon_mail_to` with `$alert_mail_message`. |
| **terminateCron_successful_completion** | Reset status flags to **0**, mark job as *Success*, record run time, write final log entries and exit 0. |
| **terminateCron_Unsuccessful_completion** | Log failure, invoke `email_and_SDP_ticket_triggering_step`, exit 1. |
| **email_and_SDP_ticket_triggering_step** | Mark job status *Failure*, send a detailed SDP ticket e‑mail (with CC list) if a ticket has not already been created, update status file. |
| **Main driver block** | Prevents concurrent runs via PID check, reads the current flag from the status file, and executes the appropriate subset of functions to resume from the last successful step. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Recon.properties` – defines all `$MNAAS_*` variables (paths, filenames, mail addresses, Java JAR location, Hive/Impala hosts, etc.). |
| **External Services** | • **HDFS** – `hdfs dfs -put`, `-rm`, `chmod` <br>• **Hive** – `hive -e` statements <br>• **Impala** – `impala-shell` refresh commands <br>• **Java** – `$MNAAS_Recon_JarPath` (reconciliation utility) <br>• **Mail** – `mailx` for alerts and SDP ticket <br>• **Syslog** – `logger` for audit trail |
| **File System Inputs** | • Excel recon files in `$MNAASReconStagingDir` matching `$hol_recon_file_format` and `$sng_recon_file_format` (e.g., `1.1_RAExportCDRFileSummaryReport_*.xlsx`, `SNG_RAExportCDRFileSummaryReport_*.xlsx`). <br>• Process‑status file `$MNAAS_Daily_Recon_ProcessStatusFileName`. |
| **File System Outputs** | • CSV file `$MNAAS_output_csv_path/$MNAAS_output_csv_file`. <br>• Loaded Hive tables (`*_temp`, `*_recon_report`). <br>• Archived Excel files in `$MNAASReconBackupDir`. <br>• Log file `$MNAAS_DailyRecon_LogPath`. <br>• Alert e‑mail and optional SDP ticket. |
| **Side Effects** | • Updates status file (flags, timestamps, PID). <br>• May create an SDP ticket on failure (once per run). |
| **Assumptions** | • All directories exist and are writable by the script user. <br>• Java JAR is compatible with the Excel format. <br>• Hive/Impala services are reachable and the user has required privileges. <br>• Mail relay is functional. <br>• The status file is the single source of truth for restartability. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Other MNAAS daily scripts** (e.g., `MNAAS_Daily_KYC_Feed_Loading.sh`, `MNAAS_Daily_RAReports.sh`) | Share the same property directory and often the same `$MNAAS_Daily_Recon_ProcessStatusFileName` pattern for status tracking. They may run sequentially in a nightly batch window. |
| **`MNAAS_Daily_Recon.properties`** | Central configuration; any change (paths, email addresses, JAR location) impacts this script and all sibling scripts. |
| **Java Recon JAR** (`$MNAAS_Recon_JarPath`) | Performs the heavy‑lifting conversion and report generation; used by both `MNAAS_export_files_to_csv` and `MNAAS_perform_daily_recon`. |
| **HDFS staging area** (`$MNAAS_hdfs_report_path`) | Destination for the CSV; also read by downstream reporting jobs (e.g., BI dashboards). |
| **Hive tables** (`mnaas.movenl_ra_recon_report_temp`, `mnaas.movenl_ra_recon_report`) | Populated here; downstream analytics jobs query these tables. |
| **SDP ticketing system** (via `mailx` to `insdp@tatacommunications.com`) | Failure handling integrates with the enterprise incident management process. |
| **Cron scheduler** | The script is expected to be invoked by a nightly cron entry; the PID guard prevents overlapping runs. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed Excel files** | Process aborts early, alerts sent. | Add a pre‑run validation step that checks file count and optionally validates schema (e.g., using `xlsx2csv` dry‑run). |
| **Java JAR failure (exit ≠ 0)** | No CSV generated → downstream tables empty. | Capture JAR stdout/stderr to a dedicated log, monitor exit codes, and implement a retry with exponential back‑off. |
| **HDFS permission or space issues** | `hdfs dfs -put` fails, job stops. | Periodically audit HDFS quotas, and add a check for free space before the put operation. |
| **Hive/Impala metadata out‑of‑sync** | Queries return stale data. | After each load, verify table row count matches CSV line count; optionally run `invalidate metadata` in Impala. |
| **Stale PID causing false “already running”** | Job never starts. | Store PID with a timestamp; if PID older than a threshold (e.g., 4 h) assume orphaned and clear it. |
| **Alert e‑mail or SDP ticket not delivered** | Failure goes unnoticed. | Add a health‑check that verifies mail queue depth; optionally integrate with a monitoring tool (e.g., Nagios) to watch the log for “failed” entries. |
| **Hard‑coded date values (month=02, year=2025, etc.)** | Script will process wrong period when not overridden. | Replace static assignments with dynamic `date` commands; keep them only for testing. |
| **Unquoted variable expansions** (e.g., `if [ $hol_count == 0 ]`) | Word‑splitting or globbing errors. | Use `[[ ... ]]` or quote variables (`"$hol_count"`). |

---

## 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Recon.properties` is present and populated.  
   - Verify that the script user can read/write all directories referenced in the properties (staging, backup, log, HDFS).  
   - Confirm Java, Hive, Impala, and mail utilities are in the `$PATH`.  

2. **Typical Invocation** (via cron or manually)  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Daily_Recon.sh
   ```

3. **Manual Debug Mode**  
   - Enable Bash tracing (already set with `set -x`).  
   - Run with `bash -x` to see each command and expanded variables.  
   - After a failure, inspect the log file `$MNAAS_DailyRecon_LogPath`.  
   - Check the status file `$MNAAS_Daily_Recon_ProcessStatusFileName` to see the last flag and PID.  

4. **Recovering from a Partial Run**  
   - Identify the current flag (`grep MNAAS_Daily_ProcessStatusFlag` in the status file).  
   - If the flag is **2‑6**, re‑run the script; it will resume from the next step automatically.  
   - To force a fresh start, set the flag to `0` (or delete the status file) **only after confirming no other instance is running**.  

5. **Testing with Sample Data**  
   - Place a pair of test Excel files matching the expected naming pattern into `$MNAASReconStagingDir`.  
   - Run the script in a sandbox environment and verify:  
     * CSV is created in `$MNAAS_output_csv_path`.  
     * HDFS path contains the CSV with correct permissions.  
     * Hive temp table receives the rows (`SELECT COUNT(*) FROM mnaas.movenl_ra_recon_report_temp`).  
     * Production table receives a new partition (`SHOW PARTITIONS mnaas.movenl_ra_recon_report`).  

---

## 7. External Config / Environment Variables  

| Variable (populated in `.properties`) | Purpose |
|---------------------------------------|---------|
| `MNAAS_Daily_Recon_ProcessStatusFileName` | Path to the status file that stores flags, PID, timestamps, job status, etc. |
| `MNAASReconStagingDir` | Directory where incoming Excel recon files land. |
| `recon_file_format`, `hol_recon_file_format`, `sng_recon_file_format` | Glob patterns for file discovery. |
| `MNAASReconBackupDir` | Archive location for processed Excel files. |
| `MNAAS_output_csv_path`, `MNAAS_output_csv_file` | Destination for the generated CSV. |
| `MNAAS_Recon_JarPath`, `MNAAS_Java_Property_File`, `MNAAS_Java_Log_File` | Java utility and its configuration. |
| `load_process_name`, `recon_process_name` | Arguments passed to the Java JAR to indicate which operation to perform. |
| `MNAAS_hdfs_report_path` | HDFS folder where the CSV is uploaded. |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `MNAAS_DailyRecon_LogPath` | Central log file for this script. |
| `recon_mail_to`, `export_mail_from` | Email addresses for alert notifications. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Recipients / metadata for SDP ticket e‑mail. |
| `GenevaLogPath` | Additional log used when generating previous‑month reports. |

If any of these variables are missing or point to non‑existent locations, the script will fail early (often with a `logger` entry).  

---

## 8. Suggested Improvements (TODO)

1. **Replace hard‑coded date values**  
   ```bash
   # Current static assignments (for testing only)
   month=$(date +"%m")
   year=$(date +"%Y")
   prevYear=$((year-1))
   prevMon=$(date +'%m' -d 'last month')
   day=$(date +"%d")
   ```
   This will make the script production‑ready and eliminate manual edits.

2. **Add robust error handling around external commands**  
   - Wrap `hdfs dfs`, `hive`, and `impala-shell` calls in functions that capture exit codes, log detailed output, and optionally retry.  
   - Example: `run_hdfs_put()` that checks free space before `-put` and retries up to 3 times.

--- 

*End of documentation.*