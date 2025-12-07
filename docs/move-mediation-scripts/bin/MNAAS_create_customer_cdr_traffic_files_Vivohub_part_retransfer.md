**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_traffic_files_Vivohub_part_retransfer.sh`

---

## 1. High‑Level Summary
This Bash driver processes a limited set of “traffic” files that have been staged for a **partial re‑transfer** of Vivohub data. For each configured feed it picks the newest N files, applies per‑customer filter conditions (via `awk`), prefixes the filename with the customer identifier, writes the filtered rows to a **backup directory**, normalises line endings, sets permissive file modes, and generates a checksum file. Execution status, PID, and success/failure flags are recorded in a shared *process‑status* file; on failure an SDP ticket and notification email are raised. The script is guarded against concurrent runs and is normally launched by a scheduler (cron).

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `traffic_file_creation` | Core loop: iterates over configured feeds, selects files, runs per‑customer `awk` filtering, writes to backup location, cleans CR characters, sets permissions, creates checksum files, and logs progress. |
| `terminateCron_successful_completion` | Resets the process‑status flag to *idle*, records success metadata (run time, job status), logs completion, and exits with status 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, invokes ticket/email step (if needed), and exits with status 1. |
| `email_and_SDP_ticket_triggering_step` | Updates the status file to *Failure*, checks whether an SDP ticket has already been raised, and if not sends a templated email to the support mailbox (via `mailx`). |
| Main block (bottom of script) | Guard against concurrent execution using the PID stored in the status file, decides whether to run `traffic_file_creation` based on the flag value, and drives the termination functions. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files.properties` (defines associative arrays, path variables, `No_of_files_to_process`, delimiters, email address, etc.)<br>- Environment variables referenced indirectly via the property file (e.g., `MNAASLocalLogPath`, `MNAASConfPath`, `MNAAS_Customer_temp_Dir`). |
| **Input Files** | - Source staging directory: `/backup1/MNAAS/Files_to_be_transferred/SNG_03Oct_Partial_retransfer`<br>- Files matching patterns in `FEED_FILE_PATTERN` (e.g., `*.txt`, `*.csv`). |
| **Output Files** | - Backup files: `<BackupDir>/<customer‑backup‑subdir>/<customer><delimiter><original‑filename>`<br>- Checksum files: same path with suffix defined by `sem_delimiter` (e.g., `.ck`).<br>- Log file: `<MNAASLocalLogPath>/MNAAS_Create_Customer_cdr_traffic_files_Vivohub_part_retransfer.log_YYYY‑MM‑DD` (stderr redirected).<br>- Process‑status file: `<MNAASConfPath>/MNAAS_Create_Customer_cdr_traffic_files_Vivohub_part_retransfer_ProcessStatusFile`. |
| **Side‑Effects** | - Creates missing backup sub‑directories (`mkdir -p`).<br>- Alters file permissions to `777` (read/write/execute for all).<br>- Updates the shared status file (flags, PID, timestamps).<br>- May send an email & create an SDP ticket on failure. |
| **Assumptions** | - The property file correctly defines all associative arrays (`FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING`, `MNAAS_Create_Customer_traffic_files_filter_condition`, `MNAAS_Create_Customer_traffic_files_backup_dir`, etc.).<br>- `awk` column positions and filter expressions are syntactically valid for the input files.<br>- The script runs on a host with sufficient disk space in both staging and backup locations.<br>- `mailx` is configured and the `SDP_ticket_from_email` variable is defined. |

---

## 4. Interaction with Other Components

| Component | Relationship |
|-----------|--------------|
| **Other “traffic” scripts** (`MNAAS_create_customer_cdr_traffic_files.sh`, `MNAAS_create_customer_cdr_traffic_files_*.sh`) | Share the same property file and process‑status file naming convention; they may run on the same schedule but guard against concurrent execution via the PID flag. |
| **Process‑status file** (`*_ProcessStatusFile`) | Central coordination point used by multiple scripts to indicate running state, success/failure, and to prevent overlapping runs. |
| **Scheduler (cron)** | Invokes this script (typically nightly) with the same environment as other mediation drivers. |
| **Backup storage** (`/backup1/MNAAS/Partfiles_SNG_03_Oct`) | Consumed downstream by downstream ingestion pipelines or manual audit processes. |
| **SDP ticketing system** | Triggered via email to `insdp@tatacommunications.com` when the script fails; other scripts use the same email template and ticketing flow. |
| **Logging infrastructure** | Log file is written to a common log directory (`MNAASLocalLogPath`) and may be harvested by log aggregation tools (e.g., Splunk, ELK). |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – two instances could corrupt the status file or duplicate processing. | Data duplication / missed files. | Guard is already in place; ensure the status file is on a reliable shared filesystem and that the PID is cleared on abnormal termination (e.g., add a trap for `EXIT`). |
| **Incorrect/empty associative arrays** – missing entries cause script to skip customers or generate malformed `awk` commands. | Missing customer files, silent data loss. | Validate the property file at script start (e.g., check required keys exist) and fail fast with a clear log entry. |
| **Permission 777 on backup files** – security exposure. | Unauthorized read/write. | Review requirement; if not needed, tighten to `664` (or appropriate) and restrict execution user. |
| **Disk‑space exhaustion** in backup directory. | Job aborts, incomplete files. | Add pre‑run disk‑space check; monitor directory usage via alerts. |
| **`awk` syntax errors** due to malformed filter conditions. | Script exits with non‑zero status, tickets raised. | Unit‑test filter expressions on a sample file; log the generated command before `eval`. |
| **Mailx failure** – ticket not raised. | Failure goes unnoticed. | Capture mailx exit code; if non‑zero, write an additional entry to a “failed‑ticket” log for manual follow‑up. |

---

## 6. Running / Debugging the Script

1. **Manual invocation** (for testing):  
   ```bash
   export MNAASCreateCustomerCdrTrafficScriptName=MNAAS_create_customer_cdr_traffic_files_Vivohub_part_retransfer.sh
   ./MNAAS_create_customer_cdr_traffic_files_Vivohub_part_retransfer.sh
   ```
   Ensure the user has read access to the staging dir and write access to the backup dir.

2. **Check prerequisites** before running:  
   - Verify the property file exists and is readable.  
   - Confirm required associative arrays contain entries for the intended feed(s).  
   - Ensure `mailx` is functional (test with a simple mail).  

3. **Debugging tips**:  
   - The script starts with `set -x`; the full command trace is appended to the log file.  
   - Tail the log while the job runs: `tail -f $MNAAS_Create_Customer_cdr_traffic_files_LogPath`.  
   - After a run, inspect the status file to see the flag values and timestamps.  
   - If the script exits early, check the exit code (`echo $?`) and the last log lines for “failed” messages.  

4. **Cron integration**:  
   - The typical crontab entry mirrors other mediation drivers, e.g.:  
     ```
     02 02 * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_create_customer_cdr_traffic_files_Vivohub_part_retransfer.sh >> /dev/null 2>&1
     ```
   - Ensure the cron environment loads the same PATH and any required module loads as an interactive shell.

---

## 7. External Configurations & Variables

| Variable / File | Purpose |
|-----------------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files.properties` | Central property file defining feed patterns, customer mappings, filter conditions, directory paths, delimiters, `No_of_files_to_process`, and email settings. |
| `MNAASLocalLogPath` | Base directory for log files. |
| `MNAASConfPath` | Directory where the process‑status file resides. |
| `MNAASCreateCustomerCdrTrafficScriptName` (set implicitly by the script name) | Used in log messages and status updates. |
| `SDP_ticket_from_email` | Sender address for failure notification emails. |
| `delimiter`, `sem_delimiter` | Characters used to build new filenames and checksum filenames. |
| `CUSTOMER_MAPPING` (associative array) | Determines whether a customer is “active” (`0`) and should be processed. |
| `FEED_FILE_PATTERN`, `FEED_CUSTOMER_MAPPING` | Map feed identifiers to filename glob patterns and to the list of customers that consume each feed. |
| `MNAAS_Create_Customer_traffic_files_filter_condition` | `awk` boolean expression per customer. |
| `MNAAS_Create_Customer_traffic_files_column_position` | Column list for `awk` output per customer. |
| `MNAAS_Create_Customer_traffic_files_backup_dir` | Sub‑directory under `BackupDir` for each customer. |
| `No_of_files_to_process` | Upper bound on how many newest files per feed are handled in a single run. |

---

## 8. Suggested Improvements (TODO)

1. **Add robust validation of the property file** – before entering the main loop, script should verify that all required associative arrays contain the expected keys for each feed and customer, and abort with a clear error if any are missing.

2. **Replace permissive `chmod 777` with a configurable, least‑privilege mode** – expose a variable (e.g., `FILE_PERM`) in the property file and default to `664` (or a value matching the organization’s security policy). This reduces exposure while preserving required access for downstream processes.