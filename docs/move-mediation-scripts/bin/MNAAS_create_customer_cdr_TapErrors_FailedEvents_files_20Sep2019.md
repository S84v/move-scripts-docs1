**High‑Level Summary**  
`MNAAS_create_customer_cdr_TapErrors_FailedEvents_files_20Sep2019.sh` is a Bash driver that processes two classes of “Tap” data produced by the Mediation layer: (1) Tap‑Error records and (2) Failed‑Event records.  For each incoming file found in the daily staging directory (`$MNAASMainStagingDirDaily_v2`) the script filters rows per‑customer using `awk` expressions defined in a property file, writes the filtered rows to customer‑specific output directories, creates matching backup copies, generates checksum files, updates file permissions, and finally moves the original source file to an archive staging area (`$MNAASMainStagingDirDaily_v1`).  Execution status, PID, and success/failure flags are recorded in a shared *process‑status* file; on failure (in PROD) an SDP ticket and notification email are raised.  The script is invoked by a scheduler (cron) and contains self‑guarding logic to avoid concurrent runs.  

---

## 1. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| `tap_errors_file_creation` | *[currently not invoked]* Reads Tap‑Error feed patterns, iterates over up‑to‑50 newest files, applies per‑customer `awk` filter (`$condition`) and column selection (`${MNAAS_Create_Customer_taperrors_files_column_position[$customer]}`), writes filtered data to customer‑specific output and backup dirs, normalises line endings, sets `777` permissions, creates checksum (`cksum`) files, and archives the source file. |
| `failed_events_file_creation` | Handles “Failed Events” files (identified by `$Failed_Events_extn`). For each file (max 50 newest) creates checksum files in both the customer “Regulatory” dir and its backup, copies the file to the regulatory dir, sets permissions, and archives the source file. |
| `terminateCron_successful_completion` | Writes a *success* flag (`0`) and job metadata (run time, status) to the process‑status file, logs completion, and exits `0`. |
| `terminateCron_Unsuccessful_completion` | Logs failure, triggers `email_and_SDP_ticket_triggering_step` when `ENV_MODE=PROD`, and exits `1`. |
| `email_and_SDP_ticket_triggering_step` | Updates the status file to *Failure*, checks whether an SDP ticket has already been raised, and if not sends a templated email (via `mailx`) to the support mailbox and CC list. Updates the status file to mark the ticket/email as sent. |
| **Main block** | Prevents concurrent execution by checking the PID stored in the status file, updates the PID, reads the current flag (`Create_Customer_TapErrors_FailedEvents_flag`), and decides which processing function(s) to run. In the current version only `failed_events_file_creation` is executed; on success the script calls `terminateCron_successful_completion`. |

---

## 2. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.properties` – defines: <br>• Directory roots (`$MNAASMainStagingDirDaily_v2`, `$MNAASMainStagingDirDaily_v1`, `$customer_dir`, `$BackupDir`) <br>• Associative arrays for feed patterns, customer mappings, filter conditions, column positions, output & backup sub‑dirs (`FEED_FILE_PATTERN_TAPERRORS`, `FEED_CUSTOMER_MAPPING_TAPERRORS`, `MNAAS_Create_Customer_taperrors_files_filter_condition`, etc.) <br>• Delimiters (`$delimiter`, `$sem_delimiter`) <br>• Log and status file paths (`$MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath`, `$MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename`) <br>• Email settings (`$SDP_ticket_from_email`) |
| **Environment variables** | `ENV_MODE` (e.g., `PROD`/`DEV`) – controls ticket/email generation. |
| **Runtime inputs** | Files matching patterns in `$MNAASMainStagingDirDaily_v2` (Tap‑Error files and Failed‑Event files). |
| **Primary outputs** | • Customer‑specific filtered Tap‑Error files (`$customer_dir/<out_dir>/<customer><delimiter><orig_name>`) <br>• Corresponding backup files (`$BackupDir/<backup_dir>/…`) <br>• Checksum files (`*.cksum`) for both output and backup <br>• Archived original files moved to `$MNAASMainStagingDirDaily_v1` <br>• Updated process‑status file (flags, PID, timestamps, job status) <br>• Log entries appended to `$MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath` <br>• Optional SDP ticket email (via `mailx`). |
| **Side effects** | • File system modifications (create, move, chmod) <br>• Potentially large I/O if many files are present <br>• Email generation (external SMTP) <br>• Process‑status file is a coordination point for other scripts. |
| **Assumptions** | • Bash 4+ (associative arrays) <br>• All directories exist and are writable by the script user <br>• `awk`, `sed`, `cksum`, `mailx` are available on the host <br>• Property file correctly defines all referenced associative arrays and variables <br>• No other process modifies the same staging directories concurrently (guarded only by PID check). |

---

## 3. Interaction with Other Components  

| Component | Relationship |
|-----------|--------------|
| **Up‑stream Mediation scripts** | Produce Tap‑Error and Failed‑Event files in `$MNAASMainStagingDirDaily_v2`. |
| **Down‑stream customer ingestion jobs** | Consume the per‑customer output files generated under `$customer_dir/...` (e.g., loading into Hive/Impala, further ETL). |
| **Backup/Retention scripts** (e.g., `MNAAS_backup_file_retention.sh`) | May archive or purge the backup directories populated by this script. |
| **Process‑status monitor** (`MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename`) | Shared with other “create_customer_cdr” scripts to coordinate flags and prevent overlapping runs. |
| **Alerting/SDP ticketing system** | Email sent via `mailx` to `insdp@tatacommunications.com` triggers an SDP ticket; downstream ticket‑handling processes act on it. |
| **Scheduler (cron)** | Executes the script on a defined schedule (typically nightly). |
| **Logging infrastructure** | Log file path is consumed by log‑aggregation tools (e.g., Splunk, ELK). |

---

## 4. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed property file** | Script aborts, no files processed, downstream pipelines stall. | Validate existence and syntax of the properties file at start; exit with clear error code; alert via monitoring. |
| **Empty associative arrays / undefined keys** | `awk` command may be built with empty filter or column list → malformed output or runtime error. | Add defensive checks (`[[ -z ${array[$key]} ]] && log_error && continue`). |
| **File permission errors** (chmod, write) | Files not readable/writable → job failure, data loss. | Ensure the script runs under a dedicated service account with proper ACLs; pre‑flight permission test. |
| **Concurrent runs** (PID check race) | Two instances could process the same files, causing duplicate work or file loss. | Use a lock file (`flock`) instead of PID‑only check; make lock acquisition atomic. |
| **Large number of files** (more than 50) | Only first 50 processed; remaining files linger, causing backlog. | Parameterise the batch size; add monitoring for stale files; consider processing all files or looping until directory empty. |
| **`awk` syntax errors** (due to special characters in filter) | Script exits with non‑zero status, no checksum generated. | Escape filter strings when building the command; test filters on sample data. |
| **Email/SMTP outage** (ticket not raised) | Failure not notified to support. | Capture `mailx` exit status; fallback to writing a local alert file; integrate with monitoring system. |
| **Checksum mismatch later** (corrupted copy) | Downstream validation may fail. | Verify checksum after copy (compare source vs. backup) and log discrepancy. |

---

## 5. Running & Debugging the Script  

1. **Prerequisites** – Ensure the property file exists at `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.properties` and all directories referenced inside are present and writable.  
2. **Manual execution** (for testing):  
   ```bash
   export ENV_MODE=DEV   # or PROD
   ./MNAAS_create_customer_cdr_TapErrors_FailedEvents_files_20Sep2019.sh
   ```
3. **Enable verbose tracing** – uncomment the `set -x` line near the top or invoke with `bash -x script.sh` to see each command as it runs.  
4. **Check logs** – tail the log file defined by `$MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath` to verify start/end timestamps and any error messages.  
5. **Validate process‑status file** – after a run, inspect `$MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename` for the flag (`Create_Customer_TapErrors_FailedEvents_flag`) and PID.  
6. **Simulate failure** – force an error (e.g., remove write permission on a backup dir) and confirm that the script writes a failure entry, sends an email (if `ENV_MODE=PROD`), and exits with status 1.  
7. **Unit‑test filter logic** – extract a sample Tap‑Error file and run the generated `awk` command manually to ensure the expected rows/columns are produced.  

---

## 6. External Config / Environment References  

| Reference | Purpose |
|-----------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.properties` | Central configuration: directory paths, associative arrays for feed patterns, customer mappings, filter conditions, column positions, delimiters, log & status file locations, email settings. |
| `ENV_MODE` (environment variable) | Determines whether SDP ticket/email is generated on failure (`PROD` triggers, other values skip). |
| `mailx` configuration (system‑wide) | Required for sending SDP ticket emails; must have a working SMTP relay. |
| `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename` (path defined in properties) | Shared status/heartbeat file used for concurrency control and flag management across the MNAAS “create_customer_cdr” family of scripts. |
| `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath` (path defined in properties) | Central log file for this script; monitored by ops. |

---

## 7. Suggested Improvements (TODO)

1. **Replace PID‑only concurrency guard with `flock`** – create a lock file (e.g., `/var/lock/MNAAS_TapErrors_FailedEvents.lock`) and acquire an exclusive lock at script start; this eliminates race conditions and stale PID issues.  
2. **Parameterise batch size & make file iteration robust** – expose a configurable `MAX_FILES_PER_RUN` (default 50) via the properties file, and loop until the staging directory is empty rather than silently leaving excess files unprocessed.  

---