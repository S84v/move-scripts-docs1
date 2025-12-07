**MNAAS_SimInventory_backup_files_process.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

This script orchestrates the end‑to‑end handling of daily SimInventory backup files for the MNAAS mediation platform. It discovers newly‑generated backup, empty, and rejected CSV files, removes duplicate rows, records file‑level metadata (size, record count, customer type, timestamps) into a control file, loads that metadata into a Hive staging table and then into a partitioned production table, copies the original CSVs (gzipped) to a remote edge‑node via SCP, and finally cleans up the local copies. Throughout the run it updates a shared process‑status file, writes detailed logs, and on failure raises an SDP ticket and email notification.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **siminventory_backup_files_to_process** | Scan the *daily* SimInventory backup directory, de‑duplicate each CSV, compute file metrics, and append a “NonEmpty” record line to the control file `$MNAAS_backup_filename_siminventory`. |
| **siminventory_empty_files_to_process** | Process files that are known to be empty (size 0). Writes a “Empty” entry to the control file and creates a zero‑size placeholder record. |
| **siminventory_reject_files_to_process** | Process files that were rejected by upstream validation. Writes a “Reject” entry to the control file. |
| **load_backup_files_records_to_temp_table** | Remove any previous HDFS staging file, copy the control file to HDFS, truncate the Hive staging table `$siminventory_file_record_count_inter_tblname`, and load the HDFS file into that table. |
| **insert_data_to_mainTable** | Insert staged rows into the partitioned production Hive table `$siminventory_file_record_count_tblname` (dynamic partitioning). Refreshes the corresponding Impala view via `impala-shell`. |
| **siminventory_SCP_backup_files_to_edgenode2** | Read the control file line‑by‑line, gzip the appropriate source file (regular, empty, or rejected) and SCP it to the remote edge‑node (IP = 192.168.124.28) preserving the original directory structure. |
| **siminventory_remove_backup_files** | After successful transfer, delete the local source files (regular, empty, or rejected) to free space. |
| **terminateCron_successful_completion** | Set the process‑status flag to *Success* (0), write final timestamps, and exit 0. |
| **terminateCron_Unsuccessful_completion** | Log failure, invoke `email_and_SDP_ticket_triggering_step`, and exit 1. |
| **email_and_SDP_ticket_triggering_step** | Mark job status as *Failure*, send an email to the MOVE‑DEV team, and raise an SDP ticket via `mailx` (Cloudera Support). |

The **main driver** at the bottom checks the current flag in `$MNAAS_SimInventory_files_backup_ProcessStatusFile` and executes the appropriate subset of the above functions, allowing the job to resume from the last successful step after a crash or manual restart.

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_ShellScript.properties` – defines all path variables, Hive/Impala DB names, file extensions, number of files to process, log locations, email lists, etc. |
| **External services** | Hadoop HDFS (`hadoop fs`), Hive (`hive -e`), Impala (`impala-shell`), remote edge‑node reachable via SSH/SCP, local mail subsystem (`mail`, `mailx`). |
| **Input directories** | `$Daily_SimInventory_BackupDir/$SimInventory_extn` (regular backups) <br> `$EmptyFileDir/$SimInventory_extn` (empty files) <br> `$MNAASRejectedFilePath/$SimInventory_extn` (rejected files) |
| **Control files** | `$MNAAS_backup_filename_siminventory` – CSV‑style metadata file written by the three “*_to_process” functions. <br> `$MNAAS_SimInventory_files_backup_ProcessStatusFile` – shared status flag file updated throughout the run. |
| **Outputs** | - Log files: `$MNAAS_backup_files_logpath_Siminventory$(date +_%F)` <br> - HDFS staging file: `$MNAAS_backup_filename_hdfs_Siminventory` <br> - Hive tables: staging (`$siminventory_file_record_count_inter_tblname`) and production (`$siminventory_file_record_count_tblname`) <br> - Remote copies on edge‑node (gzipped CSVs) <br> - Clean‑up of local source files |
| **Side effects** | - Modifies the process‑status file (flags, timestamps, email‑sent flag). <br> - Sends email and creates SDP ticket on failure. <br> - Changes file permissions (`chmod 777`). |
| **Assumptions** | - All variables referenced in the properties file are defined and point to existing directories. <br> - Hadoop, Hive, Impala CLIs are in `$PATH` and the executing user has required permissions. <br> - SSH key‑based authentication is set up for `scp` to `192.168.124.28`. <br> - No spaces or special characters in file names (script uses simple `ls` parsing). |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | How it interacts |
|---------------------|------------------|
| **Upstream SimInventory producers** (e.g., `MNAAS_SimInventory_Export.sh`) | Place daily backup CSVs, empty files, and rejected files into the directories scanned by this script. |
| **Process‑status file** (`$MNAAS_SimInventory_files_backup_ProcessStatusFile`) | Shared with all MNAAS backup scripts; the flag value determines which step to start from. |
| **Hive tables** (`$dbname.$siminventory_file_record_count_inter_tblname`, `$dbname.$siminventory_file_record_count_tblname`) | Populated by this script; downstream reporting or analytics jobs read from the production table. |
| **Impala** (`$IMPALAD_HOST`) | Refreshes the Impala metadata after Hive insert. |
| **Edge‑node** (`192.168.124.28`) | Receives the gzipped CSVs for downstream consumption (e.g., archival, further processing). |
| **SDP ticketing / email** | On failure, this script triggers an email and an SDP ticket; the ticketing system consumes the generated email. |
| **Cron scheduler** | Typically invoked by a daily cron; the script guards against concurrent runs via a `ps` check. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – script checks `ps` but may still allow overlapping runs if the process name appears more than three times (e.g., other scripts). | Duplicate processing, file contention. | Use a lock file (`flock`) or PID file in a known location. |
| **Hard‑coded IP address** for the edge‑node. | Breaks when infrastructure changes. | Move IP to the properties file; optionally resolve via DNS. |
| **`ls`/`for file in \`ls …\`` parsing** – fails with spaces or newlines in filenames. | Missed files or script crash. | Replace with `find … -print0 | while IFS= read -r -d '' file; do …; done`. |
| **No retry on SCP failures** – a transient network glitch aborts the whole job. | Incomplete data transfer, downstream failures. | Add retry loop with exponential back‑off; log each attempt. |
| **`chmod 777` on directories** – overly permissive. | Security exposure. | Use least‑privilege permissions (e.g., 750) and ensure the executing user belongs to the appropriate group. |
| **Duplicate removal using `awk '!a[$0]++'` on large files** may consume excessive memory. | OOM or long runtimes. | Stream deduplication with `sort -u` (external sort) or process files in chunks. |
| **Process‑status file updates via `sed -i`** – not atomic; race conditions if multiple scripts edit simultaneously. | Corrupted status flag. | Write to a temporary file then `mv` atomically, or use `flock` around updates. |
| **No validation of environment variables** – missing or malformed paths cause silent failures. | Job aborts early, difficult to diagnose. | Add a pre‑run validation block that checks required variables and exits with a clear error. |

---

### 6. Running / Debugging the Script

**Typical invocation (cron)**  
```bash
# /etc/cron.d/mnaas_siminventory
0 2 * * *  hdfs_user  /app/hadoop_users/MNAAS/MNAAS_SimInventory_backup_files_process.sh >> /var/log/mnaas/siminventory_cron.log 2>&1
```

**Manual run (for testing / debugging)**  
```bash
# Export any missing env vars that are normally set by the properties file
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties

# Optional: force a specific start flag (e.g., start from step 3)
sed -i 's/^MNAAS_Daily_ProcessFlag=.*/MNAAS_Daily_ProcessFlag=3/' $MNAAS_SimInventory_files_backup_ProcessStatusFile

# Run with trace
bash -x /app/hadoop_users/MNAAS/MNAAS_SimInventory_backup_files_process.sh
```

**Debugging tips**

| Situation | Action |
|-----------|--------|
| Script exits early with “Process Flag doesn’t match any values” | Verify the flag file content; ensure the flag is a single integer 0‑7. |
| No files processed, but backup directory contains CSVs | Check that `$SimInventory_extn` matches the file extension (e.g., `.csv`). |
| SCP fails | Run the `scp` command manually with the same user; verify SSH keys and network connectivity to `192.168.124.28`. |
| Hive load fails | Look at the Hive log (`$MNAAS_backup_files_logpath_Siminventory…`) for syntax errors; confirm the staging table exists and has the expected schema. |
| Duplicate removal hangs | Inspect file size; consider using `sort -u` instead of the `awk` approach. |
| Email/SDP ticket not sent on failure | Verify that `mail`/`mailx` are installed and that `$ccList`, `$GTPMailId`, `$MOVE_DEV_TEAM`, `$SDP_ticket_from_email` are defined. |

---

### 7. External Configuration & Environment Variables

| Variable (defined in `MNAAS_ShellScript.properties`) | Role |
|----------------------------------------------------|------|
| `MNAAS_SimInventory_files_backup_ProcessStatusFile` | Path to the shared status flag file. |
| `MNAAS_backup_files_logpath_Siminventory` | Base path for daily log files. |
| `Daily_SimInventory_BackupDir` | Directory containing regular backup CSVs. |
| `EmptyFileDir` | Directory for empty CSVs. |
| `MNAASRejectedFilePath` | Directory for rejected CSVs. |
| `SimInventory_extn` | File extension pattern (e.g., `*.csv`). |
| `No_of_backup_files_to_process` | Max number of files to handle per run. |
| `MNAAS_backup_filename_siminventory` | Control file that accumulates metadata rows. |
| `MNAAS_backup_filename_hdfs_Siminventory` | HDFS target path for the control file. |
| `dbname`, `siminventory_file_record_count_inter_tblname`, `siminventory_file_record_count_tblname` | Hive database and table names. |
| `IMPALAD_HOST` | Hostname for Impala daemon used for refresh. |
| `ccList`, `GTPMailId`, `MOVE_DEV_TEAM`, `SDP_ticket_from_email` | Email recipients / sender for failure notifications. |
| `MNAAS_FlagValue` (derived) | Current process flag read from the status file. |

If any of these variables are missing or point to non‑existent paths, the script will fail early; a pre‑run validation step is recommended.

---

### 8. Suggested Improvements (TODO)

1. **Add a robust lock mechanism** – replace the current `ps` check with `flock` on a lock file to guarantee single‑instance execution and avoid race conditions on the status file.
2. **Externalize all host‑specific values** – move the edge‑node IP, email addresses, and any hard‑coded paths into the properties file (or a separate `environment.cfg`) and document them for easier environment promotion (DEV → PROD).  

*Optional*: Refactor the three “*_to_process” functions into a single parameterised routine to reduce code duplication (common metadata extraction logic).  

---