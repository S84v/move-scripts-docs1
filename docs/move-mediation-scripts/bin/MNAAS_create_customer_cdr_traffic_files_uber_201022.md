**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_traffic_files_uber_201022.sh`

---

## 1. High‑Level Summary
This Bash driver re‑processes “traffic” CDR files that belong to the **Uber Soft SIM** re‑processing window.  
For each feed pattern defined in the shared property file it:

1. Picks up to *N* newest files from the Uber‑specific staging directory.  
2. For every configured customer (or rating‑group sub‑customer) it creates the required output and backup directories.  
3. Filters the source file with per‑customer *awk* conditions, writes a header row, and produces a customer‑specific file in the appropriate “customer” or “Infonova” folder.  
4. Writes an identical copy to a backup location, normalises line endings, sets permissive (777) permissions, and generates a checksum file (`cksum`).  
5. Updates a shared *process‑status* file, logs detailed progress, and on failure raises an SDP ticket + email notification.  

The script is intended to be launched by a cron job and contains a PID‑guard to avoid concurrent executions.

---

## 2. Important Functions / Logical Blocks

| Name / Block | Responsibility |
|--------------|----------------|
| **`traffic_file_creation()`** | Core processing loop: iterates over feed sources, selects files, creates per‑customer output, applies filters, writes files, backups, checksums, and logs. |
| **`terminateCron_successful_completion()`** | Resets the process‑status flag, records success timestamp, writes final log entries, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion()`** | Logs failure, triggers ticket/email (via `email_and_SDP_ticket_triggering_step`), and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step()`** | Sends an SDP ticket + email (using `mailx`) the first time a failure is detected; updates status file to avoid duplicate tickets. |
| **Main script block (PID guard & flag check)** | Prevents overlapping runs, reads the *Create_Customer_traffic_files_flag* from the status file, decides whether to invoke `traffic_file_creation`. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files.properties`. Expected variables (examples):<br>• `MNAASConfPath` – directory for status files<br>• `MNAASLocalLogPath` – base log directory<br>• `MNAASCreate_Customer_cdr_traffic_scriptname` – script name used in logs<br>• `MNAAS_Customer_temp_Dir`, `customer_dir`, `Infonova_dir`, `BackupDir` – target locations<br>• `SDP_ticket_from_email`, `SDP_ticket_cc_email` – mailx parameters |
| **Associative Arrays (defined in the property file)** | • `FEED_FILE_PATTERN` – glob patterns per feed source<br>• `MNAAS_Create_Customer_traffic_files_filter_condition` – awk filter per customer<br>• `MNAAS_Create_Customer_traffic_files_column_position` – column list for awk output<br>• `MNAAS_Create_Customer_traffic_files_out_dir` / `*_backup_dir` – relative output/backup dirs<br>• `MNAAS_Create_Customer_traffic_files_columns` – header line per customer<br>• `RATING_GROUP_CUSTOMER`, `MNAAS_Sub_Customer`, `RATING_GROUP_MAPPING` – rating‑group handling |
| **Static mappings in this script** | • `FEED_CUSTOMER_MAPPING` – maps feed source “HOL_01” → `eLUX` (only active entry)<br>• `MNAAS_Customer_SECS` – maps “Uber_soft_SIM” → `SECS_41383_*` (currently unused) |
| **Input files** | Files matching `$MNAASMainStagingDirDaily_v2/$feed_file_pattern` (Uber staging dir: `/app/hadoop_users/svetukuri/Uber_soft_sim_reprocessing`). Up to `$No_of_files_to_process` (default 100) newest files are processed. |
| **Generated files** | For each customer (or sub‑customer):<br>• `<customer_dir>/<out_dir>/<customer><delimiter><sourcefilename>` (or Infonova equivalent)<br>• Same file under `<BackupDir>/<backup_dir>/`<br>• Corresponding checksum files with suffix defined by `$sem_delimiter` (e.g. `.ck`) |
| **Side‑effects** | • Directory creation (`mkdir -p`) for customer, backup, and Infonova locations.<br>• File permission changes (`chmod 777`).<br>• Log writes to `$MNAAS_Create_Customer_cdr_traffic_files_LogPath`.<br>• Update of the shared status file (`$MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename`).<br>• Potential email/SDP ticket on failure. |
| **External services** | • `mailx` for ticket/email.<br>• Underlying file system (NFS/GPFS) for staging, output, backup.<br>• No direct DB, queue, or SFTP calls in this script (other scripts may consume the generated files). |
| **Assumptions** | • All associative arrays referenced are defined in the sourced property file.<br>• The staging directory contains files with the expected delimiter (`;`).<br>• The script runs on a host with required utilities (`awk`, `sed`, `cksum`, `mailx`).<br>• The process‑status file exists and is writable. |

---

## 4. Interaction with Other Components

| Component | How this script connects |
|-----------|--------------------------|
| **`MNAAS_create_customer_cdr_traffic_files.properties`** | Provides all feed patterns, filter conditions, column mappings, directory roots, and email parameters. |
| **Process‑status file** (`$MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename`) | Shared with other “traffic” scripts; flag `Create_Customer_traffic_files_flag` controls execution, PID guard prevents overlap, status fields (`MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_job_ran_time`) are updated. |
| **Cron scheduler** | The script is invoked by a cron entry (not shown). The PID guard ensures only one instance runs per schedule. |
| **Down‑stream ingestion pipelines** | Generated per‑customer files are consumed by downstream mediation or billing systems (e.g., Hadoop jobs, ETL pipelines). The naming convention (`<customer><delimiter><original>`), checksum files, and directory layout are expected by those pipelines. |
| **Ticketing / Notification system** | On failure, the script sends an SDP ticket via `mailx` to `insdp@tatacommunications.com`. Other scripts may monitor the same mailbox for alerts. |
| **Potential sibling scripts** | Scripts such as `MNAAS_create_customer_cdr_traffic_files_retransfer.sh` or the generic `MNAAS_create_customer_cdr_traffic_files.sh` share the same property file and status file; they may run on different schedules for other customers or re‑transfer windows. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing/incorrect associative arrays** (e.g., `FEED_FILE_PATTERN` not defined) | Script aborts with “unbound variable” or empty loops, leading to no output. | Add validation at start: `[[ -z ${FEED_FILE_PATTERN+x} ]] && echo "Missing FEED_FILE_PATTERN" && exit 1`. |
| **Hard‑coded `chmod 777`** | Overly permissive files may breach security policies. | Replace with least‑privilege mode (e.g., `chmod 664`) and make mode configurable via a property. |
| **Variable assignment typo** (`No_of_files_to_process = 100`) | Bash treats this as a command, leaving the variable empty → `head -` receives no limit. | Fix to `No_of_files_to_process=100` and add `declare -i No_of_files_to_process`. |
| **Large file volumes** (awk on multi‑GB files) | High CPU / memory, possible OOM. | Profile awk commands, consider streaming (`awk ... >> file`) already used; optionally split files or use `gawk` with `--posix`. |
| **Concurrent runs** (PID guard race condition) | Two instances could start if the status file is edited between checks. | Use `flock` on a lockfile instead of manual PID check. |
| **Mailx failure** (SMTP down) | Failure notification not sent, operators unaware. | Capture mailx exit code; if non‑zero, write to a separate “alert” file and/or trigger a secondary alert (e.g., syslog). |
| **Checksum generation errors** (cksum not found) | Missing checksum leads to downstream validation failures. | Verify `cksum` existence at script start; fallback to `md5sum` if unavailable. |
| **Path/permission errors on shared NFS** | Files not written, job stalls. | Pre‑run a health‑check that all target directories are writable; log and abort early if not. |

---

## 6. Running / Debugging the Script

### Normal Execution (cron)

```bash
# Cron entry (example, run nightly at 02:30)
30 2 * * * /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/MNAAS_create_customer_cdr_traffic_files_uber_201022.sh >> /dev/null 2>&1
```

The script will:

1. Source the property file.
2. Acquire the PID guard.
3. Check the `Create_Customer_traffic_files_flag` (0 or 1) in the status file.
4. Process up to 100 newest files per feed.
5. Update the status file to “Success” and exit.

### Manual Run (for testing / debugging)

```bash
# Export any missing env vars that the property file expects
export MNAASConfPath=/app/hadoop_users/MNAAS/conf
export MNAASLocalLogPath=/app/hadoop_users/MNAAS/logs

# Run with Bash debug enabled
bash -x /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/MNAAS_create_customer_cdr_traffic_files_uber_201022.sh
```

*Tips:*

- **Check the status file** before and after: `cat $MNAASConfPath/MNAAS_Create_Customer_cdr_traffic_files_uber_ProcessStatusFile`.
- **Tail the log** while the script runs: `tail -f $MNAASLocalLogPath/MNAAS_Create_Customer_cdr_traffic_files_uber.log_$(date +%F)`.
- **Force a failure** (e.g., rename a required directory) to verify ticket/email generation.
- **Validate output**: after run, confirm files exist under `$customer_dir` and `$BackupDir` with correct checksum files.

---

## 7. External Config / Environment Dependencies

| Item | Purpose | Typical Location |
|------|---------|------------------|
| `MNAAS_create_customer_cdr_traffic_files.properties` | Defines feed patterns, filter conditions, column mappings, directory roots, email parameters. | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` |
| `MNAASConfPath` | Directory containing the process‑status file. | Set in property file or exported by the scheduler. |
| `MNAASLocalLogPath` | Base path for log files. | Set in property file. |
| `MNAASCreate_Customer_cdr_traffic_scriptname` | Script name used in logs/status. | Usually set in property file (`MNAAS_create_customer_cdr_traffic_files_uber_201022.sh`). |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` | Email addresses for ticket generation. | Property file. |
| `customer_dir`, `Infonova_dir`, `BackupDir`, `MNAAS_Customer_temp_Dir` | Root directories for output, backup, and temporary files. | Property file. |
| `delimiter`, `sem_delimiter` | Delimiters used in filenames and checksum suffixes. | Property file. |
| `mailx` | Sends SDP ticket email. | System binary, must be in `$PATH`. |
| `awk`, `sed`, `cksum`, `chmod`, `mkdir`, `find` | Core Unix utilities. | System packages. |

If any of these variables are missing, the script will abort or behave unpredictably; therefore a pre‑flight check is advisable.

---

## 8. Suggested Improvements (TODO)

1. **Fix variable assignment & add strict mode**  
   ```bash
   set -euo pipefail
   No_of_files_to_process=100   # remove spaces
   ```
   This prevents silent failures and ensures required variables are defined.

2. **Replace manual PID guard with `flock`**  
   ```bash
   exec 200>/var/lock/MNAAS_traffic_uber.lock
   flock -n 200 || { logger "Another instance is running"; exit 1; }
   ```
   Guarantees atomic lock acquisition and eliminates race conditions.

*Additional optional enhancements:* make permission mode configurable, externalize the checksum command, and add a summary CSV of processed files for downstream audit.