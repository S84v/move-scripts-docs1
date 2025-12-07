**High‑Level Documentation – `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh`**  

---

### 1. Purpose (one‑paragraph summary)  
This Bash driver extracts *Tap‑Error* and *Failed‑Events* CDR files from the daily staging area, filters and reshapes them per‑customer according to configurable rules, writes the transformed records to customer‑specific output directories, creates backup copies, generates checksum files, and updates a shared process‑status file. It is invoked by a scheduler (cron) and contains self‑guarding logic to prevent concurrent runs, detailed logging, and automatic failure notification via email/SDP ticket when executed in production.

---

### 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`tap_errors_file_creation`** | Loops over configured Tap‑Error feed sources, selects up to 50 newest files, applies per‑customer `awk` filter/column selection, writes transformed files to customer and backup dirs, normalises line endings, sets permissions, creates checksum files, and moves the source file to the “v1” archive directory. |
| **`failed_events_file_creation`** | Similar to the above but processes *Failed‑Events* feeds (single newest file per source). Copies the raw file to a temporary customer dir and a backup dir, generates checksums, then moves the transformed file and checksum into the customer output directory. |
| **`terminateCron_successful_completion`** | Writes a *success* flag, job status, timestamp, and “email not created” flag to the shared process‑status file; logs completion and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, triggers `email_and_SDP_ticket_triggering_step` when `ENV_MODE=PROD`, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates the status file to *Failure*, checks whether an SDP ticket/email has already been sent, and if not, composes a `mailx` message to the internal ticketing address (`insdp@tatacommunications.com`). Updates the status file to record that the ticket/email was created. |
| **Main script block** | Prevents parallel execution by checking the PID stored in the status file, reads the current *process flag* (`Create_Customer_TapErrors_FailedEvents_flag`), and decides which creation routine to run (currently only `failed_events_file_creation`). On normal exit it calls the success terminator; on any error it calls the failure terminator. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.properties` – defines directories (`MNAASMainStagingDirDaily_v2`, `MNAASMainStagingDirDaily_v1`, `customer_dir`, `BackupDir`, etc.), feed‑to‑file‑pattern maps (`FEED_FILE_PATTERN_TAPERRORS`, `FEED_FILE_PATTERN_FAILEDEVENTS`), per‑customer column positions, filter conditions, output sub‑dirs, delimiters, and the path to the *process‑status* file (`MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename`). |
| **Environment variables** | `ENV_MODE` (used to decide whether to send SDP tickets), `SDP_ticket_from_email`, `SDP_ticket_cc_email`. |
| **File system – inputs** | Files matching the configured glob patterns under `$MNAASMainStagingDirDaily_v2/` (Tap‑Error and Failed‑Events feeds). |
| **File system – outputs** | • Per‑customer transformed files (`$customer_dir/<out_dir>/<customer><delimiter><orig_name>`) <br>• Backup copies in `$BackupDir/<backup_dir>/` <br>• Checksum files (`cksum` output) with suffix defined by `$sem_delimiter` <br>• Archived source files moved to `$MNAASMainStagingDirDaily_v1/` <br>• Updated process‑status file (flags, timestamps, job status). |
| **External services** | `logger` (syslog), `mailx` (SDP ticket/email), `cksum` (checksum utility). |
| **Side effects** | File permission changes (`chmod 777`), removal of Windows carriage returns (`sed -e "s/\r//g"`), possible creation of duplicate files if the same source is processed twice (guarded by PID check). |
| **Assumptions** | • All required directories exist and are writable by the script user. <br>• The properties file defines all associative arrays used (`declare -A`). <br>• `awk` filter expressions are syntactically correct for the input CSV format. <br>• `mailx` is configured to reach the internal ticketing system. |

---

### 4. Integration Points (how this script connects to the rest of the system)  

| Connection | Description |
|------------|-------------|
| **Process‑status file** (`*_ProcessStatusFilename`) | Shared with many other MNAAS scripts (e.g., `MNAAS_create_customer_cdr_NonMove_traffic_files.sh`, `MNAAS_Traffic_seq_check.sh`). The flag values (`0,1,2`) drive the execution path of this script and are updated by other scripts to indicate overall pipeline state. |
| **Staging directories** (`MNAASMainStagingDirDaily_v2` / `v1`) | Populated by upstream ingestion jobs (e.g., network element collectors). Other scripts may also read from or write to these locations. |
| **Customer output directories** (`$customer_dir`) | Consumed downstream by reporting, billing, or analytics pipelines. The naming convention (`<customer><delimiter><filename>`) matches the expectations of downstream loaders. |
| **Backup directory** (`$BackupDir`) | Used by archival/retention scripts such as `MNAAS_backup_file_retention.sh`. |
| **Cron scheduler** | The script is typically invoked from a cron entry that sets `ENV_MODE` and ensures the properties file is on the path. |
| **Email/SDP ticketing** | Integrated with the internal incident management system; other scripts also call `email_and_SDP_ticket_triggering_step` when they encounter failures. |
| **Logging** | All scripts in the suite log to a common syslog facility; the log path is defined in the properties file and is monitored by operations dashboards. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID guard may fail if the status file is corrupted or the previous process hangs. | Duplicate processing, file contention, checksum mismatches. | Add a timeout check on the stored PID; if the process exceeds a reasonable runtime, clear the PID and allow a new run. |
| **Missing or malformed configuration** – associative arrays not defined leads to empty patterns or filter conditions. | No files processed, silent failures, or malformed output. | Validate the properties file at script start (e.g., test that required keys exist). Fail fast with a clear log entry. |
| **Permission errors** – `chmod 777` may be insufficient on NFS or may violate security policy. | Files not readable by downstream jobs. | Use a configurable permission mask; audit required permissions and restrict to the minimal needed (e.g., 664). |
| **Large file volumes** – Looping with `ls | head -50` may cause memory pressure or long runtimes. | Missed files, cron overlap. | Replace `ls` pipelines with `find -maxdepth 1 -type f -name "$pattern" -print0 | sort -z | head -z -n 50` or use `xargs`. |
| **Email/SDP ticket flood** – Repeated failures could generate many tickets. | Alert fatigue. | Implement a back‑off counter in the status file (e.g., only send a ticket if the last ticket was > 1 hour ago). |
| **Checksum mismatch** – If the file is altered after checksum generation, downstream validation fails. | Data integrity issues. | Ensure checksum files are generated *after* all transformations and permission changes, and that downstream jobs verify them immediately. |

---

### 6. Running / Debugging the Script  

1. **Prerequisites**  
   * Ensure the properties file `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.properties` is present and readable.  
   * Verify that all directories referenced in the properties file exist and are writable.  
   * Export `ENV_MODE` (e.g., `export ENV_MODE=PROD` for production).  

2. **Manual invocation** (for testing)  
   ```bash
   # Switch to the script directory
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files/
   # Run with bash -x for step‑by‑step tracing
   bash -x /path/to/MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh
   ```
   *The `-x` flag prints each command after expansion, useful for confirming variable values and `awk` commands.*  

3. **Checking the PID guard**  
   *Inspect the status file:*  
   ```bash
   grep -E 'MNAAS_Script_Process_Id|Create_Customer_TapErrors_FailedEvents_flag' \
        $MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename
   ```  
   *If the PID is stale, remove the line or kill the process before re‑running.*  

4. **Log inspection**  
   *All script output is appended to the log path defined in the properties file (`$MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath`).*  
   ```bash
   tail -f $MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath
   ```  

5. **Verifying output**  
   *Confirm that per‑customer files and checksum files appear in the expected directories and that the source files have been moved to the `v1` archive.*  

6. **Simulating a failure**  
   *Force an error (e.g., make the backup directory read‑only) and observe that the script logs the failure and, in PROD, sends an SDP ticket.*  

---

### 7. External Config / Environment Variables  

| Variable | Source | Use |
|----------|--------|-----|
| `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath` | properties file | Path for script‑specific log file (stderr redirected). |
| `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename` | properties file | Shared status file that stores flags, PID, timestamps, job status, and email‑sent flag. |
| `MNAASMainStagingDirDaily_v2` / `v1` | properties file | Input staging area (v2) and archive (v1). |
| `customer_dir` | properties file | Root directory for per‑customer output. |
| `BackupDir` | properties file | Directory for backup copies of transformed files. |
| `FEED_FILE_PATTERN_TAPERRORS`, `FEED_FILE_PATTERN_FAILEDEVENTS` | associative arrays in properties file | Glob patterns for locating source files per feed source. |
| `FEED_CUSTOMER_MAPPING_TAPERRORS` | properties file | Mapping of feed source → list of customers. |
| `MNAAS_Create_Customer_taperrors_files_filter_condition`, `MNAAS_Create_Customer_taperrors_files_column_position` | properties file | `awk` filter expression and column list for each customer. |
| `delimiter`, `sem_delimiter` | properties file | Delimiters used when constructing output filenames and checksum suffixes. |
| `ENV_MODE` | environment (exported before cron run) | Determines whether SDP ticket/email is sent on failure (`PROD` triggers). |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` | environment or properties file | Email headers for the ticketing system. |

---

### 8. Suggested Improvements (TODO)  

1. **Replace fragile `ls | head` pipelines** – Use `find` with `-print0` and `sort -z` to safely handle filenames containing spaces or newlines, and to avoid race conditions.  
2. **Add configuration validation** – At script start, iterate over required keys in the properties file and abort with a clear error if any are missing or empty. This will prevent silent failures when a new feed is added or a mapping is mis‑typed.  

--- 

*End of documentation.*