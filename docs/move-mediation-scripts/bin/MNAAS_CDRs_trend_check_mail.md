**File:** `move-mediation-scripts/bin/MNAAS_CDRs_trend_check_mail.sh`

---

## 1. High‑Level Summary
This script is a daily monitoring utility that inspects the backup directories on the edge node (`edgenode2`) for newly‑generated CDR (Call Detail Record) files, counts them per feed, lists the most recent files, and reports the totals for *source* CDRs as well as *empty* and *rejected* files. The results are written to three log files and emailed to a predefined distribution list. It is typically invoked by a cron job to give operations teams visibility into CDR ingestion health and to flag any abnormal spikes or missing data.

---

## 2. Key Components & Responsibilities

| Component | Type | Responsibility |
|-----------|------|-----------------|
| **Property file** `MNAAS_CDRs_trend_check_mail.properties` | External config | Declares associative arrays (`MNAAS_backup_files_edgenode2_backup_dir`, `MNAAS_backup_files_process_file_pattern`, `FEED_CUSTOMER_MAPPING`) and scalar variables (`edgenode2`, `Daily_Telena_BackupDir`). |
| `curr_dt`, `curr_month`, `display_month`, `today_date` | Variables | Date stamps used for file‑name pattern matching and email subject formatting. |
| `Source_CDR_trend_check_log`, `Customer_Source_CDR_trend_check_log`, `Reject_Empty_CDR_trend_check_log` | Log files | Destination files for the three report sections (source CDRs, customer‑source CDRs – currently unused, empty/reject files). |
| **Main nested loops** | Logic | Iterate over each backup *folder*, each *file pattern* within that folder, and each *feed* defined in `FEED_CUSTOMER_MAPPING`. |
| `ssh $edgenode2 "ls …"` | Remote command | Counts matching `.gz` files for the current date, and lists the three most recent files for each feed/folder combination. |
| `mailx -s …` | Notification | Sends the assembled logs to the operations mailing list. |
| **Empty/Rejected file section** | Logic | Counts and lists files under `/backup1/MNAAS/Customer_CDR/EmptyFiles` and `/backup1/MNAAS/Customer_CDR/Rejectedfiles` for the current month, then emails the report. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Property file (`MNAAS_CDRs_trend_check_mail.properties`).<br>• Environment variables defined therein (e.g., `edgenode2`, `Daily_Telena_BackupDir`).<br>• Remote filesystem state on `edgenode2` (backup directories, empty/reject folders). |
| **Outputs** | • Three log files under `/app/hadoop_users/MNAAS/MNAASCronLogs/`.<br>• Two email messages (source CDR trend, empty/reject trend) sent via `mailx`. |
| **Side Effects** | • SSH connections to `edgenode2` (requires password‑less SSH keys).<br>• Remote `ls` commands that read directory metadata.<br>• Email generation (depends on local `mailx` configuration). |
| **Assumptions** | • The property file correctly defines all associative arrays and scalar variables.<br>• The edge node is reachable and the backup directories exist with the expected naming convention (`${feed}_${file}.gz`).<br>• `mailx` is installed and can relay to the corporate mail system.<br>• Sufficient disk space for log files; log rotation is handled elsewhere. |

---

## 4. Integration Points & System Connections

| Connected Component | Interaction |
|---------------------|-------------|
| **Other “MNAAS” scripts** (e.g., `MNAAS_Actives_tbl_Load.sh`, `MNAAS_Billing_Export.sh`) | These scripts generate and move CDR files into the backup directories that this script monitors. |
| **Cron scheduler** | The script is intended to run daily (typically via a cron entry under the `MNAAS` user). |
| **Edge node (`edgenode2`)** | Remote host where the actual CDR backup files reside; all file‑counting operations are performed via SSH. |
| **Mail distribution list** | Recipients include operations, development, and business contacts; the list is hard‑coded in the script. |
| **Logging infrastructure** | Log files are stored in a shared Hadoop user directory; downstream log‑aggregation tools may ingest them. |
| **Potential downstream alerting** | If integrated with a monitoring platform (e.g., Splunk, ELK), the emailed reports could be parsed for automated alerts. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **SSH connectivity failure** (node down, key rotation) | No report generated; teams lose visibility. | – Verify SSH key health daily.<br>– Add retry logic and exit codes.<br>– Alert on SSH failure before email step. |
| **Missing or malformed property file** | Script aborts, logs empty. | – Validate required variables at start (e.g., `[[ -z $edgenode2 ]] && exit 1`).<br>– Version‑control property files with change‑audit. |
| **Large directory listings** (many files) causing long `ls` or timeout | Delayed cron run, possible overlap with next schedule. | – Use `find -maxdepth 1 -type f -name "*${curr_dt}*" -printf "%T@ %p\n" | sort -n | tail -n 3` for efficient recent‑file extraction.<br>– Consider archiving older files. |
| **Email spamming / mailbox overflow** | Recipients may miss critical alerts. | – Implement a size limit on log files before emailing.<br>– Use a single consolidated report or attach as a compressed file. |
| **Log file growth** | Disk exhaustion on the Hadoop user node. | – Rotate logs (e.g., `logrotate` with weekly compression).<br>– Purge logs older than a configurable retention period. |
| **Hard‑coded recipient list** | Maintenance overhead when personnel change. | – Externalize recipient list to a config file or environment variable. |

---

## 6. Running & Debugging the Script

1. **Manual execution** (as the `MNAAS` user):
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_CDRs_trend_check_mail.sh
   ```
   The script already starts with `set -x`, so each command will be echoed to stdout for tracing.

2. **Check exit status**:
   ```bash
   echo $?   # 0 = success, non‑zero = failure
   ```

3. **Inspect generated logs**:
   ```bash
   less /app/hadoop_users/MNAAS/MNAASCronLogs/MNAAS_CDRs_trend_check_mail.log
   less /app/hadoop_users/MNAAS/MNAASCronLogs/Source_MNAAS_CDRs_trend_check_mail.log
   less /app/hadoop_users/MNAAS/MNAASCronLogs/MNAAS_Reject_Empty_CDRs_trend_check_mail.log
   ```

4. **Validate remote access**:
   ```bash
   ssh $edgenode2 "ls $Daily_Telena_BackupDir"
   ```

5. **Test email delivery** (without remote commands):
   ```bash
   echo "Test body" | mailx -s "MNAAS Test" your.email@domain.com
   ```

6. **Enable verbose SSH** (if troubleshooting connectivity):
   ```bash
   export SSH_OPTS="-vvv"
   ssh $edgenode2 $SSH_OPTS "ls ..."
   ```

---

## 7. External Configurations & Dependencies

| Item | Purpose |
|------|---------|
| `MNAAS_CDRs_trend_check_mail.properties` | Defines associative arrays for backup directories, file patterns, feed‑to‑customer mapping, and scalar variables (`edgenode2`, `Daily_Telena_BackupDir`). |
| Environment variables (implicitly sourced) | May include `PATH`, `MAILX` configuration, SSH options. |
| `mailx` binary | Sends the final reports; relies on system MTA (e.g., sendmail, postfix). |
| SSH key pair for `edgenode2` | Enables password‑less remote commands. |
| Directory structure on `edgenode2` | `/backup1/MNAAS/Customer_CDR/EmptyFiles/` and `/backup1/MNAAS/Customer_CDR/Rejectedfiles/` must exist. |

---

## 8. Suggested Improvements (TODO)

1. **Add robust error handling** – wrap each SSH/ls call in a function that checks the exit code, logs failures, and optionally aborts or continues based on severity.
2. **Externalize recipient list & log rotation** – move email addresses and log‑file paths to a separate properties file or environment variable, and integrate with `logrotate` to prevent uncontrolled growth. 

---