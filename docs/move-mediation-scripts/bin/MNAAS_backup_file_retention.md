**File:** `move-mediation-scripts/bin/MNAAS_backup_file_retention.sh`  

---

## 1. High‑Level Summary
This script enforces per‑customer backup‑file retention policies on the Edge‑2 server. For each customer and each defined process (file‑type), it SSH‑connects to the remote backup directory, finds files older than the configured retention period, and deletes up to a configurable batch size (default = 100 files). Execution status, timestamps, and any failure notifications are written to a shared process‑status file and a dedicated log file. If a failure occurs, an SDP ticket is raised via `mailx`. The script is intended to run under cron and includes a self‑guard against concurrent executions.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_backup_files_retention_delete_file`** | Loops through all customers (`${!MNAAS_file_retention_period[@]}`) and all process names (`${!MNAAS_dtail_extn[@]}`), builds the remote path, runs an `ssh find` to list files older than the configured retention period, and deletes up to `$No_of_files_to_process`. Updates the process‑status file with the current function name and logs each step. |
| **`terminateCron_successful_completion`** | Marks the job as successful in the shared status file (`MNAAS_job_status=Success`), records run time, resets the process‑status flag, logs completion, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Sets `MNAAS_job_status=Failure` in the status file, checks whether an SDP ticket/email has already been generated, and if not, sends a templated email via `mailx` (which creates an SDP ticket) and updates the flag to avoid duplicate tickets. |
| **Main Program (bottom block)** | Prevents concurrent runs by checking the PID stored in the status file, updates the PID, validates the process‑status flag (0 or 1), calls the delete routine, and finally invokes the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Property_Files/MNAAS_backup_file_retention.properties` – defines:<br>• `$MNAAS_file_retention_period_logpath` (log file)<br>• `$MNAAS_file_retention_period_ProcessStatusFileName` (shared status file)<br>• `$MNAAS_edge2_backup_path`, `$MNAAS_edge2_customer_backup_dir` (remote backup root & per‑customer sub‑dirs)<br>• `$MNAAS_edge2_user_name`, `$MNAAS_edge_server_name` (SSH credentials)<br>• Associative arrays `MNAAS_file_retention_period[customer]` (days) and `MNAAS_dtail_extn[process]` (file‑name pattern) |
| **Environment / Globals** | `$MNAAS_backup_file_retention_scriptName` (script identifier used in status file), `$SDP_ticket_from_email`, `$MOVE_DEV_TEAM`, `$SDP_Receipient_List` – used for ticket/email generation. |
| **External Services** | • **SSH** to `${MNAAS_edge2_user_name}@${MNAAS_edge_server_name}` (requires key‑based auth).<br>• **mailx** for SDP ticket/email.<br>• **logger** (syslog) for audit trail. |
| **Outputs** | • Log entries appended to `$MNAAS_file_retention_period_logpath`.<br>• Updated `$MNAAS_file_retention_period_ProcessStatusFileName` (flags, timestamps, job status, email‑sent flag).<br>• Remote file deletions on the Edge‑2 server. |
| **Assumptions** | • All associative arrays are correctly populated in the properties file.<br>• The executing user has SSH key access and permission to delete files on the remote path.<br>• `mailx` is configured and can reach the SDP ticketing system.<br>• The status file is writable and not corrupted. |

---

## 4. Interaction with Other Scripts & Components  

| Connected Component | How the Connection Occurs |
|---------------------|---------------------------|
| **Other “MNAAS_*” scripts** | All scripts share the same *process‑status* file (`$MNAAS_file_retention_period_ProcessStatusFileName`). They read/write flags (`MNAAS_backup_files_retention_ProcessStatusFlag`, `MNAAS_Script_Process_Id`, etc.) to coordinate execution and to surface job health to monitoring dashboards. |
| **Cron Scheduler** | Typically invoked by a nightly/early‑morning cron entry (e.g., `0 2 * * * /path/MNAAS_backup_file_retention.sh`). The PID guard prevents overlapping runs caused by delayed previous executions. |
| **Edge‑2 Backup Server** | Remote file system accessed via SSH; the script assumes the directory layout defined by `$MNAAS_edge2_backup_path` and per‑customer sub‑dirs. |
| **SDP Ticketing / Email** | Failure handling uses `mailx` to send a formatted email that is ingested by the SDP ticketing system. The same email address list (`$MOVE_DEV_TEAM`) is used across the suite for alerting. |
| **Logging / Monitoring** | `logger -s` writes to syslog; external monitoring tools (e.g., Splunk, ELK) may ingest these entries. The status file is also read by dashboard scripts that display job health. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Accidental deletion of needed files** (wrong retention period or pattern) | • Add a *dry‑run* mode (`--dry-run`) that only lists files.<br>• Validate retention values (>0) before issuing `ssh rm`.<br>• Keep a configurable backup of the file list (e.g., write to a “to‑delete” audit file). |
| **SSH connectivity failure** (network outage, key rotation) | • Retry logic with exponential back‑off.<br>• Alert immediately if `ssh find` returns non‑zero.<br>• Ensure key rotation process updates the script’s known user. |
| **Concurrent execution leading to race conditions** | The PID guard already prevents overlap; verify that the status file is atomically updated (use `flock` if needed). |
| **Status file corruption** (partial writes) | • Write to a temporary file then `mv` into place.<br>• Periodic health‑check script to verify required keys exist. |
| **SDP ticket spam** (multiple failures in short time) | • The flag `MNAAS_email_sdp_created` prevents duplicate tickets per run; consider a throttling window (e.g., only one ticket per hour per script). |
| **Log growth** (unbounded log file) | Rotate logs via `logrotate` or implement size‑based truncation within the script. |

---

## 6. Typical Run / Debug Workflow  

1. **Preparation**  
   ```bash
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_backup_file_retention.properties
   # Verify associative arrays are populated
   declare -p MNAAS_file_retention_period MNAAS_dtail_extn
   ```
2. **Dry‑run (manual check)** – add a temporary `echo` before the `ssh rm` line or run the `find` command directly:  
   ```bash
   ssh ${MNAAS_edge2_user_name}@${MNAAS_edge_server_name} \
       "find $customer_backup_file_path -type f -name ${file_format}* -mtime +${retention_period} | tail -$No_of_files_to_process"
   ```
3. **Execute the script** (as the intended cron user):  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_backup_file_retention.sh
   ```
4. **Monitor**  
   - Tail the log: `tail -f $MNAAS_file_retention_period_logpath`  
   - Check the status file for flags and timestamps.  
   - Verify remote deletion: `ssh ... "ls -l $customer_backup_file_path | wc -l"` before/after.  
5. **Debugging**  
   - The script runs with `set -vx`; re‑run manually to see expanded commands.  
   - If the script exits with status 1, inspect the log for “file removal failed” or “ssh find command failed”.  
   - Use `ps -fp <PID>` to ensure no stray processes remain.  

---

## 7. External Configuration & Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `MNAAS_backup_file_retention.properties` | Central configuration – defines paths, SSH credentials, associative arrays for retention periods and file extensions, log and status file locations. |
| `$MNAAS_file_retention_period_logpath` | Full path to the script‑specific log file (used for `exec 2>>`). |
| `$MNAAS_file_retention_period_ProcessStatusFileName` | Shared status file where flags, PID, job status, and email‑sent flag are stored. |
| `$MNAAS_edge2_user_name`, `$MNAAS_edge_server_name` | SSH user and host for the Edge‑2 backup server. |
| `$MNAAS_edge2_backup_path`, `$MNAAS_edge2_customer_backup_dir` | Base directory and per‑customer sub‑directory on the remote server. |
| `$SDP_ticket_from_email`, `$MOVE_DEV_TEAM`, `$SDP_Receipient_List` | Email fields used when generating an SDP ticket on failure. |
| `$MNAAS_backup_file_retention_scriptName` | Identifier written into the status file (set in the properties file). |
| `$No_of_files_to_process` (hard‑coded = 100) | Maximum number of files to delete per customer/process per run. |

---

## 8. Suggested Improvements (TODO)

1. **Add a dry‑run / preview mode** – a command‑line flag (`-n` or `--dry-run`) that logs the files that *would* be deleted without invoking `ssh rm`. This reduces risk of accidental data loss.
2. **Externalize the batch size** – move `No_of_files_to_process` into the properties file (or accept as an argument) so that different environments can tune the deletion window without editing the script.

---