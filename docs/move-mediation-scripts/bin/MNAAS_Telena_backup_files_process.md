**MNAAS_Telena_backup_files_process.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Telena_backup_files_process.sh` orchestrates the end‑to‑end handling of Telena backup files produced by the mediation layer. It scans configured directories for new backup files, de‑duplicates them, extracts file‑level metrics, enriches the records with customer and partition information, writes a semicolon‑delimited manifest, loads the manifest into a Hive staging table (via HDFS), merges the data into the production Hive table, copies the original files to a remote edge node (SCP + gzip), and finally removes or archives the local copies. Throughout the run it updates a shared process‑status file, writes detailed logs, and raises an SDP ticket + email on failure.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **telena_backup_files_to_process** | - Marks process start in status file.<br>- Iterates over configured file‑type patterns and processing locations.<br>- De‑duplicates each file (`awk '!a[$0]++'`).<br>- Computes size, record count, average record length.<br>- Derives country/instance, timestamp, partition, and customer mapping.<br>- Appends a line to the manifest `$MNAAS_backup_filename_telena`. |
| **load_backup_files_records_to_temp_table** | - Updates status flag to *2*.<br>- Copies the manifest to HDFS.<br>- Truncates the Hive staging table `$telena_file_record_count_inter_tblname`.<br>- Loads the HDFS file into the staging table. |
| **insert_data_to_mainTable** | - Updates status flag to *3*.<br>- Executes a Hive `INSERT … SELECT` with dynamic partitioning into the production table `$telena_file_record_count_tblname`.<br>- Refreshes the Impala metadata (`impala-shell -q "$telena_file_record_count_refresh"`). |
| **telena_SCP_backup_files_to_edgenode2** | - Updates status flag to *4*.<br>- Generates a unique list of files from the manifest.<br>- Gzips each file, then (in PROD) SCPs it to the remote backup server.<br>- Logs success/failure per file. |
| **telena_remove_backup_files** | - Updates status flag to *5*.<br>- Deletes the local copies (or, in non‑PROD, removes files older than 40 days). |
| **terminateCron_successful_completion** | - Resets status flags to *0* (idle) and marks job status *Success*.<br>- Writes final log entries and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | - Logs failure, triggers ticket/email, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | - Sets job status *Failure*.<br>- Sends an email to the support mailbox and marks the SDP ticket flag to avoid duplicate alerts. |
| **Main program block** | - Prevents concurrent runs via PID stored in the status file.<br>- Reads the current flag and executes the functions in the appropriate order to resume a partially‑completed run. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Telena_backup_files_process.properties` – defines all path variables, associative arrays (`MNAAS_backup_files_process_file_pattern`, `MNAAS_File_processed_location_dir`, `FEED_CUSTOMER_MAPPING`, `MNAAS_Customer_SECS`, `MNAAS_telena_type_location_mapping`), HDFS/Hive table names, log file path, backup server host, etc. |
| **Environment variables** | `ENV_MODE` (PROD / non‑PROD), `IMPALAD_HOST`, `dbname`, `MNAAS_telena_backup_files_Scriptname` (script name for logging). |
| **External services** | Hadoop HDFS, Hive, Impala, remote SSH/SCP backup server, local Linux logger, `mailx` for email alerts. |
| **Primary input files** | Backup files matching patterns under directories referenced by `MNAAS_File_processed_location_dir` (e.g. `.../Empty`, `.../Reject`, etc.). |
| **Generated files** | - Manifest: `$MNAAS_backup_filename_telena` (semicolon‑delimited).<br>- Temporary manifest: `$MNAAS_backup_filename_telena_temp`.<br>- Log: `$MNAAS_backup_files_logpath_Telena`.<br>- Process‑status file: `$MNAAS_Telena_files_backup_ProcessStatusFile`. |
| **HDFS side‑effects** | Removes any previous manifest (`hadoop fs -rm $MNAAS_backup_filename_hdfs_Telena/*`) and copies the new manifest into `$MNAAS_backup_filename_hdfs_Telena`. |
| **Hive/Impala side‑effects** | Truncates staging table, loads data, inserts into partitioned production table, refreshes Impala metadata. |
| **Remote side‑effects** | Copies gzipped backup files to `$backup_server:$backup_dir_location/` (only when `ENV_MODE=PROD`). |
| **Local cleanup** | Deletes or archives processed files after successful copy. |
| **Assumptions** | - All associative arrays and paths are correctly defined in the properties file.<br>- Hadoop, Hive, Impala CLIs are on the `$PATH` and the user has required permissions.<br>- SSH keys allow password‑less `scp` to the backup server.<br>- Sufficient disk space for temporary de‑duplication and manifest files.<br>- Cron runs the script with a proper umask (script later forces `chmod 777` on processed directories). |

---

### 4. Interaction with Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **Earlier “generation” scripts** (e.g., `MNAAS_Telena_files_backup_process.sh` or other mediation jobs) | Produce the raw backup files that this script consumes. |
| **Process‑status file** (`$MNAAS_Telena_files_backup_ProcessStatusFile`) | Shared across the entire Telena backup pipeline; other scripts read/write the same flags to coordinate incremental runs. |
| **Hive tables** (`$telena_file_record_count_inter_tblname`, `$telena_file_record_count_tblname`) | Populated here; downstream reporting/analytics jobs query the production table. |
| **Impala daemon** (`$IMPALAD_HOST`) | Refreshed after Hive insert to make data visible to Impala‑based consumers. |
| **Backup server** (`$backup_server`) | Remote destination for gzipped files; other archival/restore processes may pull from this location. |
| **Alerting / ticketing** (`email_and_SDP_ticket_triggering_step`) | Sends email to `Cloudera.Support@tatacommunications.com`; external ticketing system (SDP) is expected to ingest the email. |
| **Cron scheduler** | The script is intended to be invoked by a daily cron job; the PID‑check logic prevents overlapping runs. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed properties file** | Script aborts early, no processing. | Validate existence and syntax of the properties file at start; fail fast with clear log message. |
| **Large number of files exceeding `No_of_backup_files_to_process` (500)** | Some files remain unprocessed, causing backlog. | Make the limit configurable; add monitoring of the backlog size; consider processing in batches. |
| **`awk '!a[$0]++'` de‑duplication may consume excessive memory for very large files** | OOM or long runtimes. | Switch to `sort -u` pipeline or use Hadoop streaming for massive files. |
| **SCP failure (network, auth, disk full on remote)** | Files not archived, possible data loss. | Implement retry logic, verify remote free space before transfer, log checksum, and fall back to a “retry queue”. |
| **Concurrent runs (PID file stale)** | Two instances may corrupt the manifest or delete each other’s files. | Use a lockfile (`flock`) with timeout; clean up stale PID after a configurable grace period. |
| **Hive/Impala schema drift** | Insert fails, data not available downstream. | Add schema validation step before load; version the table names in the properties file. |
| **Permission changes (chmod 777)** | Security exposure. | Restrict to required user/group; replace with proper ACLs. |
| **Email/SDP ticket spam on repeated failures** | Alert fatigue. | Add a throttling mechanism (e.g., only send once per hour per failure type). |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Telena_backup_files_process.properties` is present and exported.  
   - Verify environment variables `ENV_MODE`, `IMPALAD_HOST`, `dbname` are set (e.g., `export ENV_MODE=PROD`).  
   - Confirm Hadoop, Hive, Impala CLIs are reachable (`hadoop version`, `hive --version`, `impala-shell --version`).  

2. **Manual execution**  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./MNAAS_Telena_backup_files_process.sh
   ```
   - The script runs with `set -x` so each command is echoed to the log file (`$MNAAS_backup_files_logpath_Telena`).  

3. **Checking progress**  
   - Tail the log: `tail -f $MNAAS_backup_files_logpath_Telena`.  
   - Inspect the manifest: `cat $MNAAS_backup_filename_telena`.  
   - Verify Hive tables: `hive -e "SELECT COUNT(*) FROM $dbname.$telena_file_record_count_tblname WHERE file_date = '2023-09-01';"`  

4. **Debugging failures**  
   - The script exits with `0` on success, `1` on failure.  
   - On failure, the status file will contain `MNAAS_job_status=Failure` and `MNAAS_email_sdp_created=Yes`.  
   - Review the log for the line containing `"failed"` to locate the failing function.  
   - Re‑run the script after fixing the root cause; the flag logic will resume from the appropriate step.  

5. **Cron integration**  
   - Typical crontab entry (run daily at 02:00):  
     ```cron
     0 2 * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Telena_backup_files_process.sh >> /var/log/mnaas_telena_backup.log 2>&1
     ```

---

### 7. External Config / Environment Dependencies

| Item | Usage |
|------|-------|
| **`MNAAS_Telena_backup_files_process.properties`** | Defines all directory paths, associative arrays for file patterns, customer mappings, HDFS locations, Hive/Impala table names, log file path, backup server host, etc. |
| **`ENV_MODE`** | Controls whether SCP is executed (`PROD`) or skipped (e.g., test). |
| **`IMPALAD_HOST`** | Hostname for the Impala daemon used in metadata refresh. |
| **`dbname`** | Hive database name where staging and production tables reside. |
| **`MNAAS_telena_backup_files_Scriptname`** | Human‑readable script identifier used in logs and email subject. |
| **`MNAAS_Telena_files_backup_ProcessStatusFile`** | Shared status file that stores flags, PID, job status, timestamps, and email‑sent flag. |
| **`MNAAS_backup_files_logpath_Telena`** | Path to the log file where all `logger -s` output is appended. |
| **`MNAAS_backup_filename_telena`** | Path to the manifest file generated from processed backup files. |
| **`MNAAS_backup_filename_hdfs_Telena`** | HDFS directory where the manifest is uploaded. |
| **`MNAAS_File_processed_location_dir`**, **`MNAAS_backup_files_process_file_pattern`**, **`FEED_CUSTOMER_MAPPING`**, **`MNAAS_Customer_SECS`**, **`MNAAS_telena_type_location_mapping`** | Bash associative arrays used throughout the script to resolve locations, patterns, and customer metadata. |

---

### 8. Suggested Improvements (TODO)

1. **Replace `eval` and back‑ticks with safer Bash constructs** – use `${array[key]}` directly and `$(...)` command substitution to avoid word‑splitting and injection risks.  
2. **Externalize the “max files per run” limit** (`No_of_backup_files_to_process`) to the properties file and make it configurable per environment; also add a metric (e.g., Prometheus) to monitor backlog size.  

--- 

*End of documentation.*