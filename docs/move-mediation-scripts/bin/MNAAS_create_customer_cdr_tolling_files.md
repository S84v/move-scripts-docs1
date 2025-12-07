**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_tolling_files.sh`

---

## 1. High‑Level Summary
This Bash driver processes daily “tolling” CDR files produced by the Mediation layer. For each configured feed it picks up a limited number of newest files from the staging‑v2 directory, filters and reshapes rows per‑customer according to property‑file‑defined `awk` expressions, writes the transformed records to customer‑specific output directories, creates matching backup copies, generates checksum files (except for rating‑group customers), updates file permissions, and finally archives the original source file to staging‑v1. Execution status, PID, and success/failure flags are recorded in a shared *process‑status* file; on failure an SDP ticket and email are raised. The script is intended to be launched by a scheduler (cron) and contains self‑guarding logic to avoid concurrent runs.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `tolling_file_creation` | Core loop: iterates over configured feeds, selects files, creates per‑customer temp files, applies filter conditions, writes output & backup files, generates checksums, moves files to final locations, and archives source files. |
| `terminateCron_successful_completion` | Clears the running flag, writes “Success” status, timestamps, and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, invokes ticket/email step (if needed), and exits with code 1. |
| `email_and_SDP_ticket_triggering_step` | Updates status to “Failure”, checks if an SDP ticket has already been raised, and if not sends a templated email to the SDP ticketing address. |
| **Main script block** | Checks for an existing PID, updates the PID in the status file, validates the process flag, invokes `tolling_file_creation`, and calls the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_tolling_files.properties` – defines associative arrays (`FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING`, `MNAAS_Create_Customer_tolling_files_filter_condition`, `MNAAS_Create_Customer_tolling_files_backup_dir`, `MNAAS_Create_Customer_tolling_files_out_dir`, `MNAAS_Create_Customer_tolling_files_columns`, `MNAAS_Create_Customer_tolling_files_column_position`, `CUSTOMER_MAPPING`, `RATING_GROUP_CUSTOMER`), directory paths (`MNAASMainStagingDirDaily_v2`, `MNAASMainStagingDirDaily_v1`, `BackupDir`, `customer_dir`, `MNAAS_Customer_temp_Dir`), numeric limits (`No_of_files_to_process`), delimiters (`delimiter`, `sem_delimiter`), log & status file locations (`MNAAS_Create_Customer_cdr_tolling_files_LogPath`, `MNAAS_Create_Customer_cdr_tolling_files_ProcessStatusFilename`), script name variables (`MNAAS_Create_Customer_cdr_tolling_scriptname`). |
| **Runtime environment** | Bash, `awk`, `sed`, `cksum`, `mailx`, `logger`, standard Unix utilities. |
| **External services** | Email/SMP ticketing system (`mailx` to `insdp@tatacommunications.com`). |
| **Input files** | Files matching patterns in `FEED_FILE_PATTERN` located under `$MNAASMainStagingDirDaily_v2`. |
| **Generated files** | • Per‑customer output files (`$customer_dir/${out_dir}/${customer}${delimiter}${filename}`) <br>• Backup copies (`$BackupDir/${backup_dir}/${customer}${delimiter}${filename}`) <br>• Checksum files (`*.cksum` style, named with `$sem_delimiter` suffix) <br>• Updated process‑status file (flags, PID, timestamps) <br>• Log entries appended to `$MNAAS_Create_Customer_cdr_tolling_files_LogPath`. |
| **Side effects** | Moves original source files to `$MNAASMainStagingDirDaily_v1/`. May create directories on‑the‑fly (`mkdir -p`). Alters file permissions to `777`. Sends email on failure. |
| **Assumptions** | • All associative arrays and directory variables are correctly defined in the properties file. <br>• The staging‑v2 directory contains only files matching the configured patterns. <br>• Sufficient disk space exists for backups and temp files. <br>• The process‑status file is writable by the script user. <br>• `mailx` is configured and reachable from the host. |

---

## 4. Integration Points & Connections

| Connected Component | Interaction |
|---------------------|-------------|
| **Other `MNAAS_create_customer_cdr_*.sh` scripts** | Same staging directories (`*_v1`, `*_v2`) and shared process‑status file format. They run sequentially (e.g., activation, actives, tap‑errors) and rely on the same per‑customer directory hierarchy. |
| **Mediation layer** | Produces the raw toll‑ing files placed into `$MNAASMainStagingDirDaily_v2`. |
| **Backup retention scripts** (`MNAAS_backup_file_retention.sh`, `MNAAS_backup_table.py`) | Periodically purge/retain files created by this script in the backup directories. |
| **Scheduler (cron)** | Invokes this script on a daily basis; expects the script to exit with 0 on success and non‑zero on failure for alerting. |
| **SDP ticketing system** | Receives failure notifications via the `email_and_SDP_ticket_triggering_step` function. |
| **Logging infrastructure** | Uses `logger -s` which writes to syslog; also appends to a dedicated log file. |
| **Process‑status file** (`*_ProcessStatusFilename`) | Shared with other scripts to coordinate flags, PID, and job status. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID lock** – script crashes leaving PID in status file, preventing future runs. | Job stops processing new files. | Add a sanity check on PID age (e.g., `ps -p $PID -o etime=`) and clear flag if older than a threshold. |
| **Missing/incorrect property definitions** – associative arrays or directory vars not set. | Script aborts with undefined variable errors, causing data loss. | Validate required variables at start of script; fail fast with clear log message. |
| **Permission errors** – `chmod 777` may be too permissive; directories may be owned by another user. | Security exposure or inability to write files. | Use least‑privilege permissions (`chmod 755`/`644`) and ensure script runs under dedicated service account with proper ownership. |
| **Large file volumes** – processing many files may exceed runtime limits or disk space. | Cron job overruns, backup disk fills. | Enforce `No_of_files_to_process` per run, monitor disk usage, and add alerting on directory size. |
| **Awk command injection** – filter condition strings come from properties file. | Potential command injection if properties are tampered. | Sanitize/validate filter strings; keep properties file read‑only for the service account. |
| **Checksum omission for rating‑group customers** – script skips `cksum` for those customers. | Downstream integrity checks may miss errors. | Document rationale; ensure downstream processes do not rely on missing checksums. |
| **Email/SMP failure** – `mailx` may be unavailable, causing ticket not to be raised. | Failure may go unnoticed. | Capture mailx exit status; if non‑zero, write to a fallback log and raise a system alarm. |

---

## 6. Running & Debugging the Script

1. **Standard execution (cron)**  
   ```bash
   0 2 * * * /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_create_customer_cdr_tolling_files.sh >> /var/log/mnaas/tolling_cron.log 2>&1
   ```

2. **Manual run (for testing)**  
   ```bash
   export MNAAS_ENV=dev   # if the properties file uses environment variables
   /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_create_customer_cdr_tolling_files.sh
   ```

3. **Check that only one instance runs**  
   ```bash
   ps -ef | grep MNAAS_create_customer_cdr_tolling_files.sh
   ```

4. **Inspect logs**  
   ```bash
   tail -f $MNAAS_Create_Customer_cdr_tolling_files_LogPath
   ```

5. **Validate process‑status file** (after run)  
   ```bash
   cat $MNAAS_Create_Customer_cdr_tolling_files_ProcessStatusFilename
   ```

6. **Debugging tips**  
   * Add `set -e` at the top to abort on any command failure.  
   * Use `bash -x script.sh` to get a trace of each command.  
   * Verify associative array contents by inserting `declare -p FEED_FILE_PATTERN` etc. before the main loop.  
   * Check that the temporary directory `$MNAAS_Customer_temp_Dir` is empty before each run.  

---

## 7. External Config / Environment Variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_Create_Customer_cdr_tolling_files_LogPath` | Full path to the script‑specific log file. |
| `MNAAS_Create_Customer_cdr_tolling_files_ProcessStatusFilename` | Shared status file that holds flags, PID, timestamps, and email‑ticket flags. |
| `MNAAS_Create_Customer_cdr_tolling_scriptname` | Human‑readable script identifier used in logs and status file. |
| `MNAASMainStagingDirDaily_v2` / `..._v1` | Source staging directory (v2) and archive directory (v1). |
| `BackupDir` | Root of per‑customer backup directories. |
| `customer_dir` | Root of per‑customer output directories. |
| `MNAAS_Customer_temp_Dir` | Temporary working directory for per‑customer files before final move. |
| `No_of_files_to_process` | Maximum number of files to handle per feed per run. |
| `delimiter`, `sem_delimiter` | Delimiters used to construct output filenames and checksum filenames. |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` | Email addresses used when raising an SDP ticket. |
| `RATING_GROUP_CUSTOMER` (array) | List of customers for which checksum generation is skipped. |
| `CUSTOMER_MAPPING` (assoc.) | Flag (1/0) indicating special handling for a customer. |
| `FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING` (assoc.) | Mapping of feed identifiers to filename glob patterns and to the list of customers that consume each feed. |
| `MNAAS_Create_Customer_tolling_files_filter_condition` (assoc.) | `awk` filter expression per customer. |
| `MNAAS_Create_Customer_tolling_files_columns` / `..._column_position` (assoc.) | Header definition and column selection for each customer. |

The script **sources** the properties file at the very start (`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_tolling_files.properties`), so any change to those variables requires a script restart (or a cron reload) to take effect.

---

## 8. Suggested Improvements (TODO)

1. **Add robust PID cleanup** – implement a check that clears the PID flag if the recorded PID no longer exists or is older than a configurable timeout.  
2. **Centralise error handling** – wrap the main processing loop in a function that captures any non‑zero exit status, writes a structured JSON entry to the log, and ensures `email_and_SDP_ticket_triggering_step` is always invoked on failure.  

---