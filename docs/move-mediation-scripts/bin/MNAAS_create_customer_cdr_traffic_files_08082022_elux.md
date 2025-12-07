**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_traffic_files_08082022_elux.sh`

---

## 1. High‑Level Summary
This Bash driver processes daily “traffic” CDR files produced by the Mediation layer for a specific HOL (Hall‑of‑Legends) data‑load (dated 08‑08‑2022). For each configured feed it selects up‑to‑`$No_of_files_to_process` newest files from the staging‑v2 directory, filters and reshapes the records per‑customer using `awk` expressions defined in a property file, writes the transformed rows to customer‑specific output folders (or the Infonova folder for certain customers), creates matching backup copies, normalises line endings, sets permissive file modes, generates checksum files, and finally archives the source file to a staging‑v1 “output” directory. Execution status, PID, and success/failure flags are maintained in a shared *process‑status* file; on failure an SDP ticket and email are raised. The script is intended to be invoked by a scheduler (cron) and contains self‑guarding logic to prevent concurrent runs.

---

## 2. Key Functions / Logical Blocks

| Function / Block | Responsibility |
|------------------|----------------|
| **`traffic_file_creation`** | Core processing loop: iterates over configured feed sources, selects files, creates per‑customer directories, runs `awk` filters, writes transformed data to output & backup locations, normalises CR/LF, sets permissions, creates checksum (`cksum`) files, and moves the original file to the archive directory. |
| **`terminateCron_successful_completion`** | Clears the “running” flag, writes *Success* status, timestamps, and logs a normal termination message. |
| **`terminateCron_Unsuccessful_completion`** | Logs an abnormal termination and exits with a non‑zero status. |
| **`email_and_SDP_ticket_triggering_step`** | On first failure, updates the status file to *Failure*, composes an SDP ticket email (via `mailx`) with log and status file references, and marks the ticket as sent to avoid duplicate alerts. |
| **Main script block** | PID guard (checks `$MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename` for a running PID), reads the process flag, invokes `traffic_file_creation` when allowed, and routes to the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_traffic_files.properties` – defines associative arrays (`FEED_FILE_PATTERN`, `MNAAS_Create_Customer_traffic_files_filter_condition`, `MNAAS_Create_Customer_traffic_files_out_dir`, `MNAAS_Create_Customer_traffic_files_backup_dir`, `CUSTOMER_MAPPING`, `RATING_GROUP_CUSTOMER`, `RATING_GROUP_MAPPING`, `MNAAS_Sub_Customer`, etc.) and scalar variables (`MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename`, `MNAAS_Create_Customer_cdr_traffic_scriptname`, `MNAAS_FlagValue`, `No_of_files_to_process`, `delimiter`, `sem_delimiter`, `MNAAS_Customer_temp_Dir`, `customer_dir`, `Infonova_dir`, `BackupDir`, `SDP_ticket_from_email`, `SDP_ticket_cc_email`, …). |
| **Environment / Globals** | `set -x` (debug trace), `logger` (syslog), `mailx` (email), `awk`, `sed`, `cksum`, `chmod`, `mkdir`, `find`, `mv`. |
| **Primary Input Files** | Files matching patterns in `$MNAASMainStagingDirDaily_v2` (e.g. `$feed_file_pattern`). The script processes up to `$No_of_files_to_process` newest files per feed. |
| **Generated Output Files** | For each customer: <br>• Transformed traffic file (`$customer_dir/.../$newfilename`) <br>• Backup copy (`$BackupDir/.../$newfilename`) <br>• Corresponding checksum files (`*.cksum` with `$sem_delimiter` suffix) <br>• Optional temporary files in `$MNAAS_Customer_temp_Dir` (removed after move). |
| **Side Effects** | • Directory creation (`mkdir -p`). <br>• File permission changes (`chmod 777`). <br>• Source file move to `$MNAASMainStagingDirDaily_v1`. <br>• Updates to the shared *process‑status* file (flags, PID, timestamps, job status). <br>• Syslog entries via `logger`. <br>• SDP ticket email on failure. |
| **Assumptions** | • All associative arrays and scalar variables are correctly defined in the sourced properties file. <br>• The staging directories exist and are readable/writable by the script user. <br>• `awk` field positions and filter conditions are valid for the input file format (semicolon‑delimited). <br>• The system has sufficient disk space for temporary and backup copies. <br>• `mailx` is configured to reach the internal SDP ticketing system. |

---

## 4. Integration Points & Connectivity

| Connected Component | Interaction |
|---------------------|-------------|
| **Up‑stream Mediation jobs** | Produce the raw traffic files placed in `$MNAASMainStagingDirDaily_v2`. Those jobs are typically other Bash drivers (e.g., `MNAAS_create_customer_cdr_*_files.sh`) that run earlier in the nightly pipeline. |
| **Process‑status file** (`$MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename`) | Shared with all MNAAS scripts; used for PID guarding, flag control, and overall job health reporting. |
| **Customer‑specific downstream consumers** | The output directories (`$customer_dir/...` and `$Infonova_dir/...`) are read by downstream billing, analytics, or reporting systems (often Hadoop ingestion jobs). |
| **Backup storage** (`$BackupDir`) | Serves as a safeguard for audit/re‑run; may be mirrored to a remote archive or HDFS via separate copy jobs. |
| **SDP ticketing / alerting** | On failure, `email_and_SDP_ticket_triggering_step` sends an email to `insdp@tatacommunications.com` with the log and status file references. |
| **Cron scheduler** | The script is invoked from a nightly cron entry; the PID guard prevents overlapping runs. |
| **Logging infrastructure** | Uses `logger` to write to the system syslog (or a dedicated log collector). |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Concurrent execution** – PID guard fails (stale PID) | Duplicate processing, file duplication or loss | Add a sanity check that the PID still points to a running process; if not, clear the flag before starting. |
| **Missing or malformed property definitions** (e.g., undefined arrays) | Script aborts, downstream jobs miss data | Validate required variables at start; exit with clear error if any are undefined. |
| **Disk‑space exhaustion** (temp, backup, or output dirs) | Incomplete files, checksum mismatch | Monitor free space before processing; abort early with alert if below threshold. |
| **`awk` filter syntax errors** (field positions out of range) | Empty or corrupted output files | Include a sanity check on the first few rows after `awk` to ensure non‑empty output; log warning if empty. |
| **Permission issues** (mkdir/chmod failures) | Files not readable by downstream systems | Run script under a dedicated service account with pre‑granted directory permissions; log any `chmod` failures. |
| **Email/SDP ticket failure** | No alert on failure, silent job loss | Capture `mailx` exit status; if non‑zero, write to a fallback log and raise a local alarm (e.g., write to `/var/log/cron`). |
| **Hard‑coded paths** (e.g., `/app/hadoop_users/...`) | Breakage when environment changes | Externalise all absolute paths into the properties file; enforce a naming convention. |

---

## 6. Running / Debugging the Script

1. **Standard execution (cron)**  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_create_customer_cdr_traffic_files_08082022_elux.sh
   ```
   The script logs to `$MNAAS_Create_Customer_cdr_traffic_files_LogPath` and updates the shared status file.

2. **Manual run (debug mode)**  
   ```bash
   export MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename=/tmp/status.txt
   export MNAAS_Create_Customer_cdr_traffic_scriptname=MNAAS_create_customer_cdr_traffic_files_08082022_elux.sh
   ./MNAAS_create_customer_cdr_traffic_files_08082022_elux.sh
   ```
   - `set -x` is already enabled, so each command is echoed.  
   - Tail the log while it runs: `tail -f $MNAAS_Create_Customer_cdr_traffic_files_LogPath`.

3. **Force a single feed for testing**  
   Edit the properties file to set `FEED_FILE_PATTERN` for only one feed, and set `No_of_files_to_process=1`. Verify the output and checksum files.

4. **Inspect PID guard**  
   ```bash
   grep MNAAS_Script_Process_Id $MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename
   ps -fp <PID>
   ```
   Ensure the PID belongs to this script; if stale, clear the entry manually.

5. **Check generated files**  
   ```bash
   ls -l $customer_dir/$customer
   cksum $customer_dir/$customer/*   # verify checksum files exist
   ```

6. **Simulate failure**  
   Force an `awk` error (e.g., rename a required column) and confirm that an SDP ticket email is sent and the status file reflects `Failure`.

---

## 7. External Config / Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `MNAAS_create_customer_cdr_traffic_files.properties` | Sourced at top of script | Holds all associative arrays, directory paths, delimiters, flags, and email addresses. |
| `MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename` | Property file | Path to the shared status file used for PID/flag management. |
| `MNAAS_Create_Customer_cdr_traffic_scriptname` | Property file | Script name used in logs and PID guard. |
| `MNAASMainStagingDirDaily_v2` / `v1` | Hard‑coded in script (overwrites any property) | Input staging directory (v2) and archive directory (v1). |
| `MNAAS_Create_Customer_cdr_traffic_files_LogPath` | Hard‑coded in script | Log file location. |
| `customer_dir`, `Infonova_dir`, `BackupDir`, `MNAAS_Customer_temp_Dir` | Property file | Destination directories for customer files, Infonova files, backups, and temporary work area. |
| `No_of_files_to_process` | Property file | Maximum number of files per feed to process in one run. |
| `delimiter`, `sem_delimiter` | Property file | Field separator for output filenames and checksum suffix. |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` | Property file | Email addresses used when raising an SDP ticket. |
| `FEED_FILE_PATTERN`, `MNAAS_Create_Customer_traffic_files_filter_condition`, `MNAAS_Create_Customer_traffic_files_column_position`, `MNASS_Create_Customer_traffic_files_columns`, etc. | Property file | Define per‑feed file glob, per‑customer filter expressions, column selections, and header definitions. |

---

## 8. Suggested Improvements (TODO)

1. **Add early validation of required associative arrays** – before entering the main loop, iterate over a list of mandatory keys (e.g., `FEED_FILE_PATTERN`, `MNAAS_Create_Customer_traffic_files_out_dir`, `MNAAS_Create_Customer_traffic_files_backup_dir`) and abort with a clear error if any are missing or empty.

2. **Replace hard‑coded staging paths with property‑driven variables** – move the assignments of `MNAASMainStagingDirDaily_v2` and `MNAASMainStagingDirDaily_v1` into the properties file to avoid script duplication across different date‑specific variants. This also enables a single script to be reused for multiple loads via a command‑line date argument.