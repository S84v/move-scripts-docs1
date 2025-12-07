**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_traffic_files.sh`

---

## 1. High‑Level Summary
This Bash driver extracts raw “traffic” feed files from the daily staging area (`$MNAASMainStagingDirDaily_v2`), filters and reshapes each record per‑customer business rules defined in a property file, writes the transformed rows to customer‑specific output directories, creates matching backup copies, generates checksum files, updates a shared *process‑status* file, archives the original source files, and logs every step. It is scheduled by cron, guards against concurrent executions, and on failure raises an SDP ticket and email notification.

---

## 2. Key Functions / Logical Blocks  

| Function / Block | Responsibility |
|------------------|----------------|
| **`traffic_file_creation`** | Core loop: iterates over configured feed sources, selects up‑to‑`$No_of_files_to_process` newest files, creates per‑customer output, backup, checksum files, and moves the source file to the archive directory. |
| **`terminateCron_successful_completion`** | Clears the “running” flag, writes *Success* status, timestamps, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step` (via caller), and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates the status file to *Failure*, checks the “email‑sent” flag, and if not already sent, composes an SDP ticket email via `mailx`. |
| **Main script block** | PID‑guard (prevents parallel runs), reads the process‑status flag, decides whether to invoke `traffic_file_creation`, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration (external file)** | `MNAAS_create_customer_cdr_traffic_files.properties` – defines associative arrays (`FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING`, `MNAAS_Create_Customer_traffic_files_filter_condition`, `MNAAS_Create_Customer_traffic_files_out_dir`, `MNAAS_Create_Customer_traffic_files_backup_dir`, `CUSTOMER_MAPPING`, `RATING_GROUP_CUSTOMER`, `MNAAS_Sub_Customer`, column lists, delimiters, etc.) and scalar variables (paths, flags, log file names). |
| **Environment variables** | Expected to be exported by the property file: <br>• `$MNAASMainStagingDirDaily_v2` – source staging directory (v2). <br>• `$MNAASMainStagingDirDaily_v1` – archive directory (v1). <br>• `$MNAAS_Create_Customer_cdr_traffic_files_LogPath` – log file. <br>• `$MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename` – shared status file. <br>• `$MNAAS_Create_Customer_cdr_traffic_scriptname` – script name used in status file. <br>• `$MNAAS_FlagValue` – flag read from status file. <br>• `$customer_dir`, `$Infonova_dir`, `$BackupDir`, `$MNAAS_Customer_temp_Dir` – root directories for output, Infonova, backup, temp files. |
| **Input data** | Raw traffic files matching patterns in `$FEED_FILE_PATTERN` located under `$MNAASMainStagingDirDaily_v2`. |
| **Primary outputs** | • Per‑customer traffic files (named `<customer><delimiter><original>`). <br>• Corresponding backup files (same name in `$BackupDir`). <br>• Checksum files (`cksum`) with suffix defined by `$sem_delimiter`. <br>• Updated process‑status file (flags, timestamps, job status). <br>• Log entries appended to `$MNAAS_Create_Customer_cdr_traffic_files_LogPath`. |
| **Side‑effects** | • Moves processed source files to `$MNAASMainStagingDirDaily_v1`. <br>• May generate an SDP ticket via `mailx` on failure. <br>• Creates directories on‑the‑fly (`mkdir -p`). |
| **Assumptions** | • All associative arrays are correctly populated in the property file. <br>• Required utilities (`awk`, `sed`, `cksum`, `mailx`, `logger`) are present and in `$PATH`. <br>• Files are semicolon‑delimited (or `;|,` for KYC join). <br>• Sufficient disk space for duplicate backup files. <br>• The process‑status file is writable by the script user. |

---

## 4. Interaction with Other Scripts / Components  

* **Shared Process‑Status File** – Many `MNAAS_*.sh` scripts (e.g., `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh`, `MNAAS_backup_table.py`) read/write the same status file to coordinate job sequencing and to avoid overlapping runs.  
* **Common Property Files** – The same `MNAAS_create_customer_cdr_traffic_files.properties` (or a sibling property file) is sourced by other “create_customer_cdr_*.sh” drivers, ensuring consistent directory mappings and filter conditions across the suite.  
* **Backup & Archive Scripts** – After this script finishes, downstream jobs (e.g., `MNAAS_backup_table.py`) may ingest the backup directories for long‑term retention.  
* **Email/SDP Ticketing** – The `email_and_SDP_ticket_triggering_step` uses the same SDP ticketing email address as other failure‑handling scripts, providing a unified incident‑management view.  
* **Cron Scheduler** – All `MNAAS_*.sh` drivers are invoked by a central cron table; the PID guard in this script prevents duplicate runs that could clash with other drivers that also touch the same staging directories.  

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID guard may be bypassed if the status file is corrupted or the previous PID has exited but the flag remains set. | Duplicate processing, file duplication, checksum mismatches. | Add a robust lock file (`flock`) in addition to PID check; ensure the status file is atomically updated (`sed -i` → `ed` or `awk` with temp file). |
| **Missing or malformed property entries** – Undefined associative array keys cause empty variables, leading to `mkdir -p` with empty paths or `awk` failures. | Script aborts silently, incomplete output. | Validate required keys at start of script; exit with clear error if any are missing. |
| **Large file volumes** – The script processes up to `$No_of_files_to_process` files per run, each potentially large; repeated `awk` invocations can be CPU‑intensive. | Performance degradation, missed SLA. | Consider streaming `awk` once per file (combine header + filter) or migrate to a more efficient language (Python/PySpark). |
| **Permission errors** – `chmod 777` is used indiscriminately; if the underlying filesystem enforces stricter ACLs, the command may fail. | Files left unreadable downstream. | Use explicit permission variables; log failures of `chmod` and abort if critical. |
| **Email/SDP ticket failure** – `mailx` may be unavailable, causing silent loss of incident notification. | Undetected failures. | Capture exit status of `mailx`; fallback to local log and raise a monitoring alarm. |
| **Checksum file naming** – The script builds checksum filenames by stripping the extension and appending `$sem_delimiter`; if the original file lacks an extension, the name may be malformed. | Missing checksum files. | Add defensive check for filename format; use a consistent suffix (e.g., `.cksum`). |

---

## 6. Typical Execution / Debugging Steps  

1. **Check Scheduler** – Verify the cron entry (e.g., `0 2 * * * /app/hadoop_users/MNAAS/.../MNAAS_create_customer_cdr_traffic_files.sh`).  
2. **Inspect Status File** – `grep Create_Customer_traffic_files_flag $MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename` should be `0` or `1`.  
3. **Run Manually (dry‑run)**  
   ```bash
   export MNAAS_Create_Customer_cdr_traffic_scriptname=MNAAS_create_customer_cdr_traffic_files.sh
   ./MNAAS_create_customer_cdr_traffic_files.sh   # watch stdout + log tail
   tail -f $MNAAS_Create_Customer_cdr_traffic_files_LogPath
   ```  
4. **Validate Output** – After run, confirm: <br>• Files appear under `$customer_dir/...` and `$BackupDir/...`. <br>• Corresponding `.cksum` files exist. <br>• Original files moved to `$MNAASMainStagingDirDaily_v1`. |
5. **Troubleshoot** – If the script exits with code 1: <br>• Check the log for “failed” messages. <br>• Look for `mailx` errors. <br>• Verify that all directories referenced in the property file exist and are writable. |
6. **Process‑Status Review** – After successful run, the status file should contain `MNAAS_job_status=Success` and `Create_Customer_traffic_files_flag=0`. |

---

## 7. External Config / Environment Dependencies  

| Item | Purpose |
|------|---------|
| **`MNAAS_create_customer_cdr_traffic_files.properties`** | Supplies all associative arrays and scalar variables used throughout the script (feed patterns, customer mappings, directory roots, delimiters, column lists, flag file paths). |
| **`$MNAAS_Create_Customer_cdr_traffic_files_LogPath`** | Central log file for this driver; also used by other scripts for unified logging. |
| **`$MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename`** | Shared status/heartbeat file that coordinates execution across the Move suite. |
| **`$SDP_ticket_from_email`, `$SDP_ticket_cc_email`** | Email addresses used when raising an SDP ticket on failure. |
| **Standard Unix utilities** (`awk`, `sed`, `cksum`, `logger`, `mailx`, `ps`, `grep`, `cut`, `mkdir`, `chmod`, `find`, `mv`) | Required for data transformation, file handling, and notification. |
| **`$MNAAS_FlagValue`** (read from status file) | Controls whether the script should run (0 or 1). |

---

## 8. Suggested Improvements (TODO)

1. **Refactor to Reduce Duplication & `eval` Usage**  
   * Replace the many `eval`‑wrapped `awk` commands with functions that build the command string safely and invoke `awk` directly. This eliminates the security risk of code injection and improves readability.

2. **Introduce Robust Locking & Error Handling**  
   * Use `flock -n /var/lock/MNAAS_traffic.lock` at script start to guarantee single execution.  
   * Enable `set -euo pipefail` and trap `ERR` to automatically invoke `email_and_SDP_ticket_triggering_step` on any unexpected failure, ensuring no silent exits.

---