**High‑Level Summary**  
`MNAAS_create_customer_cdr_NonMove_traffic_files.sh` is a production‑grade Bash driver that extracts “non‑move” CDR (Call Detail Record) traffic files from a daily staging area, filters and reshapes them per‑customer specifications, writes the transformed records to customer‑specific output directories, backs them up, generates checksum files, and updates a shared process‑status file. It is invoked by a scheduler (cron) and includes self‑guarding logic to avoid concurrent runs, logging, and automatic failure notification via email/SDP ticket.

---

## 1. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`nonmovecdr_file_creation`** | Main ETL loop: <br>• Reads feed‑source definitions from property arrays. <br>• Picks up to *N* newest files from `$MNAASMainStagingDirDaily_v2`. <br>• For each file, applies a per‑customer AWK filter (`condition`) and column‑selection (`column_position`). <br>• Writes transformed rows to a temporary folder, then copies to the customer’s output directory and backup directory. <br>• Normalises line endings, sets `777` permissions, creates checksum (`cksum`) files (both output and backup). <br>• Moves the original source file to `$MNAASMainStagingDirDaily_v1`. |
| **`terminateCron_successful_completion`** | Clears the “running” flag in the shared status file, marks job status *Success*, records run‑time, writes final log entries and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step` (if needed), and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | On first failure only: updates status file to *Failure*, sends an incident email (via `mailx`) to the support mailbox and creates an SDP ticket flag. Subsequent failures only log that a ticket already exists. |
| **Main script block** | Guard against concurrent execution using the PID stored in the status file. If no prior instance is running, writes its own PID, checks the process flag (`Create_Customer_nonmovecdr_files_flag`), and calls `nonmovecdr_file_creation` followed by the appropriate termination routine. |

---

## 2. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_NonMove_traffic_files.properties` – defines all path variables, associative arrays (`FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING`, `MNAAS_Create_Customer_NonMove_traffic_files_filter_condition`, `MNAAS_Create_Customer_NonMove_traffic_files_columns`, `MNAAS_Create_Customer_NonMove_traffic_files_column_position`, `MNAAS_Create_Customer_NonMove_traffic_files_out_dir`, `MNAAS_Create_Customer_NonMove_traffic_files_backup_dir`), numeric limits (`No_of_files_to_process`), delimiters (`delimiter`, `sem_delimiter`), email sender (`SDP_ticket_from_email`). |
| **Runtime environment** | Bash, `awk`, `cksum`, `mailx`, `logger`. Assumes POSIX‑compatible `ls`, `sed`, `cp`, `chmod`. |
| **Primary input files** | Files matching patterns in `$MNAASMainStagingDirDaily_v2/<pattern>` (e.g., `*.csv`). |
| **Generated output** | For each customer: <br>• Transformed CSV in `$customer_dir/${out_dir}/${customer}<delimiter><orig_name>` <br>• Backup copy in `$BackupDir/${backup_dir}/${customer}<delimiter><orig_name>` <br>• Corresponding checksum files (`*.cksum`) in both locations. |
| **Process‑status file** | `$MNAAS_create_customer_cdr_NonMove_traffic_files_ProcessStatusFilename` – holds flags, PID, job status, timestamps, email‑sent flag. Updated throughout execution. |
| **Log file** | `$MNAAS_create_customer_cdr_NonMove_traffic_files_LogPath` – appended with timestamps, progress, and error messages. |
| **Side effects** | • Moves original source file to `$MNAASMainStagingDirDaily_v1/`. <br>• Sends email (and indirectly creates an SDP ticket) on first failure. <br>• Alters file permissions to `777`. |
| **Assumptions** | • All directories exist and are writable by the script user. <br>• Property file defines every referenced associative array key for each feed/customer. <br>• No other process writes to the same status file concurrently. <br>• `mailx` is correctly configured for outbound mail. |

---

## 3. Interaction with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Up‑stream ingestion scripts** (e.g., `MNAAS_Traffic_tbl_with_nodups_loading.sh`) | Deposit raw traffic files into `$MNAASMainStagingDirDaily_v2`. |
| **Down‑stream consumption scripts** (e.g., reporting, billing, or analytics jobs) | Read the per‑customer output files produced here. |
| **Process‑monitoring framework** | Periodically reads the status file to display job health, trigger alerts, or decide whether to launch dependent jobs. |
| **Backup retention scripts** (`MNAAS_backup_file_retention.sh`) | Clean up older files in `$BackupDir`. |
| **Email/SDP ticketing system** | Receives incident notifications generated by `email_and_SDP_ticket_triggering_step`. |
| **Cron scheduler** | Executes this script on a defined schedule (typically nightly). |
| **Logging infrastructure** | Consumes the log file for aggregation (e.g., Splunk, ELK). |

---

## 4. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed property file** | Script aborts, no processing, possible stale PID lock. | Validate existence and syntax of the properties file at start; exit with clear error if required variables are unset. |
| **Concurrent runs (PID stale)** | Duplicate processing, checksum collisions, file lock contention. | Replace PID‑file guard with a robust lockfile (`flock`) and include a timeout/expiry check. |
| **Permission errors on output/backup dirs** | Files not written, downstream jobs fail. | Pre‑run `mkdir -p` with proper ACLs; fail fast if `chmod`/`cp` returns non‑zero. |
| **AWK filter syntax error** (e.g., malformed condition) | No data written, silent data loss. | Test AWK expressions during property validation; capture AWK exit status and log. |
| **Large file volume causing long runtime** | Missed cron windows, resource contention. | Add configurable `max_runtime` and split processing into batches; monitor runtime via logs. |
| **Email/SDP ticket not sent** | Failure goes unnoticed. | Verify `mailx` return code; fallback to local log entry and raise a monitoring alarm. |
| **Checksum mismatch later** | Data integrity issue. | Periodic checksum verification job; store checksum files with immutable permissions. |

---

## 5. Running / Debugging the Script  

1. **Manual invocation** (for testing):  
   ```bash
   export MNAAS_create_customer_cdr_NonMove_traffic_files_LogPath=/tmp/mnaas_nonmove.log
   ./MNAAS_create_customer_cdr_NonMove_traffic_files.sh
   ```
   Ensure the property file is present and points to test directories.

2. **Enable verbose tracing** – the script already contains `set -x`. For deeper inspection, prepend `bash -x` or temporarily add `set -eu` to abort on unset variables.

3. **Check PID lock** – if the script reports “process is running already”, inspect the status file:  
   ```bash
   cat $MNAAS_create_customer_cdr_NonMove_traffic_files_ProcessStatusFilename
   ps -fp <PID>
   ```

4. **Validate configuration** – before running, dump key variables:  
   ```bash
   echo "Staging dir: $MNAASMainStagingDirDaily_v2"
   echo "Customers for feed X: ${FEED_CUSTOMER_MAPPING["X"]}"
   ```

5. **Log inspection** – tail the log file while the job runs:  
   ```bash
   tail -f $MNAAS_create_customer_cdr_NonMove_traffic_files_LogPath
   ```

6. **Post‑run verification** – confirm that:  
   - Output files exist in the expected customer directories.  
   - Corresponding checksum files (`*.cksum`) are present.  
   - The status file shows `MNAAS_job_status=Success` and `Create_Customer_nonmovecdr_files_flag=0`.  

7. **Failure path** – if the script exits with code 1, check the log for the “Script … terminated Unsuccessfully” entry and verify that an email was sent (mail queue, `mailx -v`).  

---

## 6. External Config / Environment Variables  

| Variable (from property file) | Purpose |
|-------------------------------|---------|
| `MNAAS_create_customer_cdr_NonMove_traffic_files_LogPath` | Full path to the script‑specific log file. |
| `MNAAS_create_customer_cdr_NonMove_traffic_files_ProcessStatusFilename` | Shared status file used for flags, PID, timestamps. |
| `MNAASMainStagingDirDaily_v2` | Source directory where raw feed files land. |
| `MNAASMainStagingDirDaily_v1` | Archive directory for processed raw files. |
| `MNAAS_Customer_temp_Dir` | Temporary workspace for per‑customer transformed files. |
| `BackupDir` | Root backup location for all customers. |
| `customer_dir` | Base directory for customer‑specific output folders. |
| `FEED_FILE_PATTERN` (assoc. array) | Glob pattern per feed source (e.g., `feedA=*.csv`). |
| `FEED_CUSTOMER_MAPPING` (assoc. array) | List of customers that consume each feed. |
| `MNAAS_Create_Customer_NonMove_traffic_files_filter_condition` (assoc. array) | AWK boolean expression applied per customer. |
| `MNAAS_Create_Customer_NonMove_traffic_files_columns` (assoc. array) | Header line to write for each customer file. |
| `MNAAS_Create_Customer_NonMove_traffic_files_column_position` (assoc. array) | AWK field list (e.g., `$1,$3,$5`) for each customer. |
| `MNAAS_Create_Customer_NonMove_traffic_files_out_dir` (assoc. array) | Sub‑directory under `$customer_dir` for each customer. |
| `MNAAS_Create_Customer_NonMove_traffic_files_backup_dir` (assoc. array) | Sub‑directory under `$BackupDir` for each customer. |
| `No_of_files_to_process` | Maximum number of newest files to handle per run. |
| `delimiter`, `sem_delimiter` | String used to concatenate customer ID with original filename and to name checksum files. |
| `SDP_ticket_from_email` | Sender address used when generating the failure email. |

If any of these are missing or empty, the script will likely fail; therefore they must be validated at start‑up.

---

## 7. Suggested Improvements (TODO)

1. **Robust configuration validation** – add a function that iterates over all required variables and associative‑array keys, aborting with a clear error if any are undefined or empty. This prevents silent runtime failures.

2. **Replace PID‑file lock with `flock`** – using `exec 200>/var/run/mnaas_nonmove.lock && flock -n 200 || exit 0` provides atomic locking, avoids stale PID issues, and automatically releases the lock on script exit.

---