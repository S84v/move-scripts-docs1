**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_activation_files.sh`

---

## 1. High‑Level Summary
This Bash driver extracts “activation” CDR feed files from the daily staging‑v2 directory, filters and reshapes each file per‑customer according to rules defined in a property file, writes the transformed records to customer‑specific output folders, creates matching backup copies and checksum files, updates a shared *process‑status* file, and archives the original source files to staging‑v1.  It is scheduled by cron, protects against concurrent runs via a PID lock, logs extensively, and raises an SDP ticket/email on failure.

---

## 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `activation_file_creation` | Main processing loop: iterates over configured feed sources, selects up‑to‑`$No_of_files_to_process` newest files, applies per‑customer `awk` filters, writes transformed data to a temporary dir, backs up, sanitises line endings, sets permissions, generates `cksum` files, moves results to the customer‑specific output directory, and finally archives the source file. |
| `terminateCron_successful_completion` | Clears the *running* flag in the process‑status file, marks job status *Success*, records run time, writes final log entries and exits with status 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, invokes ticket/email step (if uncommented), exits with status 1. |
| `email_and_SDP_ticket_triggering_step` | Updates the status file to *Failure*, checks whether an SDP ticket has already been raised, and if not sends a templated email to the SDP ticketing system (`insdp@tatacommunications.com`). |
| **Main script block** | PID guard (prevents overlapping runs), reads the flag (`Create_Customer_activation_files_flag`) from the status file, decides whether to run `activation_file_creation`, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_activation_files.properties` – defines all environment variables, associative arrays, directory paths, delimiters, file patterns, customer mappings, filter conditions, column selections, etc. |
| **Environment variables** (expected to be set by the property file) | `MNAASMainStagingDirDaily_v2`, `MNAASMainStagingDirDaily_v1`, `BackupDir`, `customer_dir`, `MNAAS_Customer_temp_Dir`, `MNAAS_Create_Customer_cdr_activation_files_LogPath`, `MNAAS_Create_Customer_cdr_activation_files_ProcessStatusFilename`, `MNAAS_Create_Customer_cdr_activation_scriptname`, `SDP_ticket_from_email`, `SDP_ticket_cc_email`, `No_of_files_to_process`, `delimiter`, `sem_delimiter`, `RATING_GROUP_CUSTOMER` (array), etc. |
| **Input files** | Any file matching a pattern in `FEED_FILE_PATTERN` under `$MNAASMainStagingDirDaily_v2`.  Files are assumed to be semi‑colon (`;`) delimited text. |
| **Generated files** | For each processed customer: <br>• Transformed data file: `<customer><delimiter><original_name>` placed in `$customer_dir/${MNAAS_Create_Customer_activation_files_out_dir[$customer]}` <br>• Backup copy in `$BackupDir/${MNAAS_Create_Customer_activation_files_backup_dir[$customer]}` <br>• Checksum files (`cksum`) with suffix `$sem_delimiter` in both locations. |
| **Side effects** | • Updates the shared process‑status file (flags, PID, job status, timestamps). <br>• Writes detailed log entries to `$MNAAS_Create_Customer_cdr_activation_files_LogPath`. <br>• Moves original source files to `$MNAASMainStagingDirDaily_v1/`. <br>• May send an email/SDP ticket on failure. |
| **Assumptions** | • All associative arrays (`FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING`, `MNAAS_Create_Customer_activation_files_filter_condition`, etc.) are correctly populated in the property file. <br>• The staging, backup, and customer directories exist or are creatable by the script’s user. <br>• Files are small enough for `awk` in‑memory processing; no explicit streaming for huge files. <br>• `mailx` is configured and reachable for SDP ticket creation. |

---

## 4. Interaction with Other Scripts & Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Other “MNAAS_create_customer_cdr_*.sh” scripts** (e.g., `MNAAS_create_customer_cdr_NonMove_traffic_files.sh`, `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh`) | Share the same *process‑status* file naming convention, logging path, and staging directories. They are scheduled sequentially or in parallel by cron, each handling a different CDR feed type. |
| **Property Files** (`MNAAS_create_customer_cdr_activation_files.properties`) | Central configuration source; any change propagates to this script and other related scripts that source the same file. |
| **Backup & Archive System** | Backup directories are mirrored to a long‑term storage tier (often an HDFS or NAS mount) by downstream archival jobs not shown here. |
| **SDP Ticketing / Email System** | Failure notifications are sent via `mailx` to the internal SDP ticketing address (`insdp@tatacommunications.com`). |
| **Monitoring / Scheduler** | Cron invokes the script (typically nightly). Monitoring tools may poll the *process‑status* file for `MNAAS_job_status` and alert on “Failure”. |
| **External Utilities** | `awk`, `sed`, `cksum`, `chmod`, `mkdir`, `logger`, `ps`, `mailx`. All must be present on the host. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – PID lock may be bypassed if the status file is manually edited or the previous process becomes a zombie. | Use a lock file (`flock`) in addition to the PID check; ensure the script removes the lock on any exit path (including traps for SIGTERM/SIGINT). |
| **Missing or malformed property file** – undefined arrays cause script to abort silently. | Validate required variables/arrays at start of script (e.g., `[[ -z ${FEED_FILE_PATTERN+x} ]] && exit 1`). Add a sanity‑check function. |
| **Disk‑space exhaustion** – backup and output directories can fill quickly. | Add pre‑run disk‑usage check; integrate with a retention policy job; alert when free space < X GB. |
| **Permission errors** – `chmod 777` is overly permissive and may be blocked by security policies. | Replace with least‑privilege mode (e.g., `chmod 750`) and ensure the script runs under a dedicated service account with proper ACLs. |
| **Checksum mismatch** – generated only for non‑rating‑group customers; downstream processes may expect checksums for all. | Document the exception clearly; consider generating checksums for all files or flagging missing ones downstream. |
| **Uncaught command failures** – many commands (`mkdir`, `mv`, `awk`, `cksum`) are not checked for exit status. | Enable `set -euo pipefail` and capture return codes after critical steps; log failures before exiting. |
| **Email/SDP flood** – repeated failures could generate many tickets. | Implement a throttling counter in the status file (e.g., only send ticket if last ticket > 24 h ago). |

---

## 6. Running / Debugging the Script  

1. **Manual invocation (development)**  
   ```bash
   export MNAAS_Create_Customer_cdr_activation_scriptname=MNAAS_create_customer_cdr_activation_files.sh
   . /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_activation_files.properties
   bash -x ./MNAAS_create_customer_cdr_activation_files.sh
   ```
   *`-x`* (already set in script) prints each command as it runs.

2. **Check prerequisites**  
   - Verify that all required associative arrays are defined (`declare -p FEED_FILE_PATTERN`).  
   - Ensure staging directories contain files matching the configured patterns.  
   - Confirm write permission on log, status, backup, and customer output directories.

3. **Log inspection**  
   - Primary log: `$MNAAS_Create_Customer_cdr_activation_files_LogPath`.  
   - Process‑status file: `$MNAAS_Create_Customer_cdr_activation_files_ProcessStatusFilename` – look for `Create_Customer_activation_files_flag`, `MNAAS_job_status`, `MNAAS_job_ran_time`.

4. **Debugging tips**  
   - Insert `echo "DEBUG: <msg>"` after critical variable expansions to verify values.  
   - Run a single feed manually: `feed_source="myFeed"; file=$(ls -tr $MNAASMainStagingDirDaily_v2/${FEED_FILE_PATTERN[$feed_source]} | head -1);` then invoke the inner loop body.  
   - Use `strace -f -e trace=file,process -p <pid>` to watch file operations if the script hangs.

5. **Cron integration**  
   - Ensure the cron entry sources the same property file (or runs via a wrapper that does).  
   - Example cron line (run nightly at 02:00):  
     ```
     0 2 * * * /app/hadoop_users/MNAAS/MNAAS_create_customer_cdr_activation_files.sh >> /var/log/mnaas/cron_activation.log 2>&1
     ```

---

## 7. External Configurations & Files Referenced  

| File / Variable | Purpose |
|-----------------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_activation_files.properties` | Central configuration – defines all directory paths, associative arrays (`FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING`, filter conditions, column selections), delimiters, and operational limits (`No_of_files_to_process`). |
| `$MNAAS_Create_Customer_cdr_activation_files_LogPath` | Absolute path to the script’s log file (written via `logger -s`). |
| `$MNAAS_Create_Customer_cdr_activation_files_ProcessStatusFilename` | Shared status file that tracks PID, flags, job status, timestamps, and whether an SDP ticket has been raised. |
| `$MNAASMainStagingDirDaily_v2` / `$MNAASMainStagingDirDaily_v1` | Source staging directory (v2) and archive directory (v1). |
| `$BackupDir` | Root backup location; sub‑folders per customer are defined in the property file. |
| `$customer_dir` | Root directory for per‑customer output; sub‑folders per customer are defined in the property file. |
| `$MNAAS_Customer_temp_Dir` | Temporary workspace used to avoid checksum race conditions. |
| `$SDP_ticket_from_email`, `$SDP_ticket_cc_email` | Email addresses used when generating an SDP ticket on failure. |
| `mailx` | System utility used to send the ticket email. |

---

## 8. Suggested Improvements (TODO)

1. **Add robust error handling** – enable `set -euo pipefail`, capture exit codes after each critical command, and update the status file accordingly before exiting.
2. **Replace ad‑hoc PID lock with `flock`** – create a lock file (e.g., `/var/run/mnaas_activation.lock`) and run the whole script under `flock -n` to guarantee single‑instance execution even if the status file is manually edited.  

---