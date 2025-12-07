**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_actives_files.sh`

---

## 1. High‑Level Summary
This Bash driver extracts “actives” CDR feed files from the daily staging‑v2 directory, filters and reshapes the records per‑customer according to configurable `awk` expressions, writes the transformed rows to customer‑specific output folders, creates matching backup copies, generates checksum files (except for rating‑group customers), and finally archives the original source file to staging‑v1. Execution is guarded against concurrent runs, logs extensively, updates a shared *process‑status* file, and on failure raises an SDP ticket and email notification. The script is invoked by a scheduler (cron) in production.

---

## 2. Core Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `actives_file_creation` | Main processing loop: iterates over configured feed sources, selects up to `$No_of_files_to_process` newest files, performs per‑customer filtering/column selection via `awk`, writes temp & backup files, normalises line endings, sets permissions, creates checksums, moves results to final output, and archives the source file. |
| `terminateCron_successful_completion` | Clears the run‑flag, records success status, timestamps, and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, invokes ticket/email step (via `email_and_SDP_ticket_triggering_step`), and exits with code 1. |
| `email_and_SDP_ticket_triggering_step` | Updates the status file to *Failure*, checks if an SDP ticket has already been raised, and if not sends a templated email to the incident mailbox (using `mailx`). |
| **Main script block** | Guard against concurrent execution using the PID stored in the status file, set the run‑flag, decide whether to invoke `actives_file_creation`, and route to the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_actives_files.properties` – defines all environment variables, associative arrays, directory paths, file patterns, filter conditions, column lists, backup/out directories, delimiters, etc. |
| **External Services / Resources** | - File system (HDFS‑mounted or local POSIX) – staging directories `$MNAASMainStagingDirDaily_v2` and `$MNAASMainStagingDirDaily_v1`.<br>- Customer‑specific output directories (`$customer_dir/${MNAAS_Create_Customer_actives_files_out_dir[$customer]}`) and backup directories (`$BackupDir/...`).<br>- Log file `$MNAAS_Create_Customer_cdr_actives_files_LogPath`.<br>- Process‑status file `$MNAAS_Create_Customer_cdr_actives_files_ProcessStatusFilename`.<br>- `mailx` for SDP ticket/email.<br>- `logger` (syslog). |
| **Primary Input Files** | Files matching patterns in `FEED_FILE_PATTERN` located under `$MNAASMainStagingDirDaily_v2`. |
| **Primary Output Files** | For each processed source file:<br>• Customer‑specific transformed file (`$customer_dir/.../<customer><delimiter><original_name>`).<br>• Corresponding checksum file (`<filename>.cksum` style).<br>• Backup copy in `$BackupDir/...` with identical content and checksum.<br>• Archived source file moved to `$MNAASMainStagingDirDaily_v1/`. |
| **Side Effects** | - Updates the shared process‑status file (run flag, PID, job status, timestamps, email‑sent flag).<br>- Writes to the central log file.<br>- May generate an SDP ticket/email on failure.<br>- Creates temporary files in `$MNAAS_Customer_temp_Dir`. |
| **Assumptions** | - All required associative arrays and scalar variables are defined in the sourced properties file.<br>- The script runs on a host with Bash 4+ (associative arrays).<br>- `awk`, `sed`, `cksum`, `chmod`, `mkdir`, `mailx`, and `logger` are available in `$PATH`.<br>- The user executing the script has read/write permissions on all staging, output, backup, and temp directories.<br>- No other process manipulates the same status file concurrently. |

---

## 4. Interaction with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **Other “create_customer_cdr_…_files” scripts** (e.g., `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh`) | Share the same *process‑status* file naming convention, logging pattern, and failure‑notification mechanism. They are typically scheduled sequentially to process different feed types. |
| **Backup / Retention scripts** (`MNAAS_backup_file_retention.sh`, `MNAAS_backup_table.py`) | Operate on the same backup directories populated by this script; they enforce retention policies and archive older backups. |
| **Ticket‑generation / monitoring framework** | The `email_and_SDP_ticket_triggering_step` sends mail to `insdp@tatacommunications.com` which is consumed by the internal SDP ticketing system. |
| **Cron scheduler** | The script is invoked by a cron entry that sets the appropriate environment (e.g., `PATH`, `HOME`). |
| **Property files** (`MNAAS_create_customer_cdr_actives_files.properties`) | Centralised configuration used by multiple scripts; any change propagates to all drivers that source it. |
| **Shared temp directory** (`$MNAAS_Customer_temp_Dir`) | May be used by other scripts for intermediate processing; cleanup is implicit when files are moved to final locations. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID guard** – if the script crashes, the PID remains in the status file, preventing future runs. | Production stop for this feed. | Add a timeout check (e.g., compare start timestamp) and a “force‑run” flag; periodically clean the PID entry via a watchdog script. |
| **Missing or malformed property values** – undefined associative arrays cause Bash errors. | Script aborts, no files processed. | Validate required variables at start of script (e.g., `[[ -z $MNAASMainStagingDirDaily_v2 ]] && exit 1`). |
| **Permission errors on output/backup directories** – `mkdir` or `chmod` fails. | Files not written, downstream jobs fail. | Ensure directory ownership/ACLs are provisioned; log and raise ticket on failure. |
| **Large file volumes** – processing many files sequentially may exceed runtime windows. | Cron overlap, backlog. | Tune `$No_of_files_to_process`, parallelise per‑customer processing, or split into multiple scheduled jobs. |
| **Checksum generation skipped for rating‑group customers** – downstream validation may expect a checksum. | Inconsistent data integrity checks. | Document the exception clearly; consider generating a placeholder checksum file. |
| **Mailx failure** – SDP ticket not raised on failure. | Incident may go unnoticed. | Capture mailx exit code; fallback to a secondary notification channel (e.g., Slack webhook). |
| **Line‑ending conversion (`sed -i -e "s/\r//g"`)** – may corrupt binary files if mis‑applied. | Data corruption. | Restrict processing to known text‑based CDR formats; add file‑type guard. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_actives_files.properties` is present and readable.  
   - Verify that the staging directories (`$MNAASMainStagingDirDaily_v2`, `$MNAASMainStagingDirDaily_v1`) and the temp/backup/customer output roots exist and are writable.  
   - Confirm `mailx` is configured for the incident mailbox.

2. **Manual Execution**  
   ```bash
   # Switch to the user that runs the cron job (e.g., mnaas)
   su - mnaas
   # Optional: enable tracing
   set -x
   # Run the script
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_create_customer_cdr_actives_files.sh
   ```
   - Check the log file defined by `$MNAAS_Create_Customer_cdr_actives_files_LogPath` for start/end timestamps and any error messages.  
   - Inspect the process‑status file to confirm the flag is cleared (`Create_Customer_actives_files_flag=0`) after a successful run.

3. **Debugging Tips**  
   - **Validate configuration**: `grep -E 'FEED_FILE_PATTERN|FEED_CUSTOMER_MAPPING' $MNAAS_Create_Customer_cdr_actives_files_ProcessStatusFilename`.  
   - **Simulate a single file**: place a test file matching a known pattern in the staging‑v2 directory and run the script; verify output in the customer folder and backup dir.  
   - **Check PID guard**: `cat $MNAAS_Create_Customer_cdr_actives_files_ProcessStatusFilename | grep MNAAS_Script_Process_Id`. Kill stale PID if necessary.  
   - **Force failure path**: rename the properties file temporarily to trigger the failure branch and confirm that an SDP ticket email is sent.

---

## 7. External Config / Environment Variables

| Variable (populated in properties file) | Purpose |
|------------------------------------------|---------|
| `MNAAS_Create_Customer_cdr_actives_files_LogPath` | Full path to the script’s log file. |
| `MNAAS_Create_Customer_cdr_actives_files_ProcessStatusFilename` | Shared status file tracking flags, PID, job status, timestamps, and email‑sent flag. |
| `MNAASCreate_Customer_cdr_actives_scriptname` (derived) | Script name used in logs and status updates. |
| `MNAASMainStagingDirDaily_v2` / `..._v1` | Source staging directories (new vs. archived). |
| `FEED_FILE_PATTERN` (associative array) | Regex/filename pattern per feed source. |
| `FEED_CUSTOMER_MAPPING` (associative array) | List of customers that consume each feed source. |
| `MNAAS_Create_Customer_actives_files_filter_condition` (assoc.) | `awk` boolean expression used to filter rows per customer. |
| `MNAAS_Create_Customer_actives_files_column_position` (assoc.) | Column list for `awk` output per customer. |
| `MNASS_Create_Customer_actives_files_columns` (assoc.) | Header line to prepend to output files. |
| `MNAAS_Create_Customer_actives_files_out_dir` / `backup_dir` | Relative output and backup sub‑directories per customer. |
| `BackupDir`, `customer_dir`, `MNAAS_Customer_temp_Dir` | Root paths for backup, final output, and temporary processing. |
| `No_of_files_to_process` | Maximum number of newest files to handle per run. |
| `delimiter`, `sem_delimiter` | Delimiters used when constructing output filenames and checksum filenames. |
| `RATING_GROUP_CUSTOMER` (array) | Customers for which checksum generation is skipped. |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` | Email addresses used when raising an SDP ticket. |
| `MNAAS_email_sdp_created` flag (in status file) | Tracks whether a ticket/email has already been sent for the current failure. |

---

## 8. Suggested Improvements (TODO)

1. **Add Robust Configuration Validation**  
   Implement a `validate_config` function that checks the existence and non‑emptiness of all required variables/arrays before processing begins. Exit early with a clear error message if any are missing.

2. **Replace `sed -i` In‑Place Flag Updates with Atomic Writes**  
   Current flag manipulation (`sed -i`) can race with concurrent processes or be unsafe on NFS. Use a temporary file and `mv` to atomically replace the status file, reducing the chance of corrupted flags.

---