**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_traffic_files_retransfer_Original.sh`

---

## 1. High‑Level Summary
This Bash driver re‑processes a limited “traffic” CDR feed set for a re‑transfer window.  
It scans the daily staging directory (`/backup1/MNAAS/Files_to_be_transferred`) for files matching feed‑specific glob patterns, applies per‑customer filter expressions (defined in a property file), writes filtered rows to customer‑specific output and backup directories, normalises line endings, sets permissive permissions, generates checksum files, and records progress in a shared *process‑status* file.  The script is scheduled via cron, protects against concurrent runs with a PID guard, and raises an SDP ticket + email on failure.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `traffic_file_creation` | Core loop: for each feed pattern → list newest files → for each file → for each mapped customer → create temp filtered file, move to customer output, duplicate to backup, normalise CR/LF, chmod, generate checksum, update logs. |
| `terminateCron_successful_completion` | Clears the “running” flag, writes *Success* status, timestamps, and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, invokes ticket/email step (via `email_and_SDP_ticket_triggering_step`), exits with code 1. |
| `email_and_SDP_ticket_triggering_step` | Updates status to *Failure*, checks if an SDP ticket has already been raised, and if not sends a templated email to the SDP system (`insdp@tatacommunications.com`). |
| **Main script block** | PID‑guard logic: reads previous PID from the process‑status file, starts the job only if no live instance, updates the PID, checks the global flag (`Create_Customer_traffic_files_flag`), then calls `traffic_file_creation` and terminates appropriately. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files.properties`. Expected variables/associative arrays: <br>• `FEED_FILE_PATTERN` (feed → glob) <br>• `FEED_CUSTOMER_MAPPING` (feed → space‑separated customers) <br>• `MNAAS_Create_Customer_traffic_files_filter_condition` (customer → awk predicate) <br>• `MNAAS_Create_Customer_traffic_files_out_dir` / `backup_dir` (customer → sub‑dir) <br>• `MNAAS_Customer_temp_Dir`, `customer_dir`, `BackupDir` <br>• `MNAASLocalLogPath`, `MNAASConfPath` <br>• delimiters (`delimiter`, `sem_delimiter`) and numeric limits (`No_of_files_to_process`). |
| **Process‑status file** | `$MNAASConfPath/MNAAS_Create_Customer_cdr_traffic_files_retransfer_ProcessStatusFile` – holds PID, flag, job status, timestamps, SDP‑ticket flag. Modified throughout execution. |
| **Staging input** | Files under `$MNAASMainStagingDirDaily_v2 = /backup1/MNAAS/Files_to_be_transferred` matching each feed’s pattern. |
| **Generated files** | For each customer: <br>• Filtered CDR file (`${customer}${delimiter}${filename}`) in `customer_dir/<out_dir>/` <br>• Same file in `BackupDir/<backup_dir>/` <br>• Corresponding checksum files (`*.cksum` via `cksum`) in both locations. |
| **Side effects** | • Directory creation (`mkdir -p`). <br>• File moves (`mv -n`). <br>• Permission changes (`chmod 777`). <br>• Log entries appended to `$MNAASLocalLogPath/..._retransfer.log_YYYY-MM-DD`. <br>• System logger (`logger -s`). <br>• Potential SDP ticket/email via `mailx`. |
| **Assumptions** | • All associative arrays are defined and contain matching keys for each feed/customer. <br>• Destination directories are on a filesystem with sufficient space and allow 777 permissions. <br>• `awk`, `sed`, `cksum`, `mailx` are present and in `$PATH`. <br>• The process‑status file is writable by the script user. <br>• No other process modifies the same staging files concurrently. |

---

## 4. Integration Points & Call Flow  

1. **Up‑stream** – Mediation layer drops raw traffic CDR files into `/backup1/MNAAS/Files_to_be_transferred`.  
2. **Configuration** – The property file is shared with all “traffic” creation scripts (`*_traffic_files*.sh`). Changes to feed patterns or filter conditions affect this script automatically.  
3. **Process‑status file** – Same file is read/written by the PID‑guard logic and by other scripts that may run in the same window (e.g., `MNAAS_create_customer_cdr_traffic_files.sh`). The flag `Create_Customer_traffic_files_flag` is toggled by the orchestrator to enable/disable processing.  
4. **Down‑stream** – Customer‑specific output directories are consumed by downstream billing, analytics, or SFTP transfer jobs (not shown). The checksum files are used by those jobs to verify integrity.  
5. **Alerting** – On failure, the script sends an email that is routed to the SDP ticketing system; the ticket is then handled by the operations team.  

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing/incorrect property definitions** (e.g., undefined `FEED_FILE_PATTERN`) | Script aborts, no files processed, downstream jobs stall. | Validate property file at start (e.g., test required keys). Add unit‑test harness for the property file. |
| **Race condition on staging files** (another job moving files) | Partial processing, duplicate work, checksum mismatch. | Ensure exclusive lock via the PID guard and coordinate with any other scripts that touch the same staging directory. |
| **Large file volume causing `awk` memory pressure** | Slowdowns, possible OOM. | Add streaming (`awk` processes line‑by‑line already) but monitor memory; consider splitting files or using `gawk` with `--posix`. |
| **Permission/umask issues** (chmod 777 may be disallowed) | Files not readable by downstream systems. | Centralise permission policy; replace 777 with appropriate group/owner settings. |
| **Failed checksum generation** (cksum not found or I/O error) | Downstream integrity checks fail. | Verify `cksum` exit status; fallback to `md5sum` if needed. |
| **Email/SDP ticket flood** (multiple failures in quick succession) | Ticket overload, alert fatigue. | Debounce ticket creation using the `MNAAS_email_sdp_created` flag (already present) and add a back‑off timer. |
| **Hard‑coded paths** (`/backup1/...`) | Inflexibility across environments. | Externalise base paths into the property file or environment variables. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites** – Ensure the property file exists and is readable, the process‑status file is writable, and the staging directory contains test files matching the feed patterns.  
2. **Manual invocation** (for debugging):  
   ```bash
   export MNAASCreateCustomerCdrTrafficScriptName="MNAAS_create_customer_cdr_traffic_files_retransfer_Original.sh"
   ./MNAAS_create_customer_cdr_traffic_files_retransfer_Original.sh
   ```
   The script already runs with `set -x` (trace) and redirects `stderr` to the log file, so you will see each command printed.  
3. **Check logs** – Tail the generated log:  
   ```bash
   tail -f $MNAASLocalLogPath/MNAAS_Create_Customer_cdr_traffic_files_retransfer.log_$(date +%F)
   ```  
4. **Verify status file** – After run, inspect the process‑status file for flags, timestamps, and any `MNAAS_job_status` value.  
5. **Validate outputs** – Confirm that for each expected customer a file and its `.cksum` exist in both the output and backup directories, and that line endings are LF only (`sed -i -e "s/\r//g"`).  
6. **Simulate failure** – Force an error (e.g., remove `awk`) and verify that the SDP email is generated only once and that the status file shows `Failure`.  

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files.properties` | Supplies all directory paths, associative arrays for feed patterns, customer mappings, filter conditions, delimiters, and numeric limits. |
| `MNAASLocalLogPath` (from properties) | Base path for the per‑run log file. |
| `MNAASConfPath` (from properties) | Location of the process‑status file. |
| `MNAAS_Customer_temp_Dir`, `customer_dir`, `BackupDir` (from properties) | Temporary, final, and backup storage locations. |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` (from properties) | Email “From” and “Cc” addresses used when raising an SDP ticket. |
| `MNAAS_Create_Customer_cdr_traffic_scriptname` (set implicitly by script name) | Used in log and status messages. |
| System utilities: `awk`, `sed`, `cksum`, `mailx`, `logger`, `ps`, `grep`, `cut`, `date`, `chmod`, `mkdir`, `mv` | Core processing and notification functions. |

---

## 8. Suggested Improvements (TODO)

1. **Add explicit configuration validation** – At script start, loop over required keys in the property file and abort with a clear error if any are missing or empty.  
2. **Replace hard‑coded 777 permissions** – Introduce a configurable permission mask (e.g., `MNAAS_FILE_MODE=664`) and apply it via `chmod $MNAAS_FILE_MODE` to align with security policies.  

---