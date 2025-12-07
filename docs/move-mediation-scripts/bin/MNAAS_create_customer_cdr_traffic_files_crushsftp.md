**File:** `move-mediation-scripts/bin/MNAAS_create_customer_cdr_traffic_files_crushsftp.sh`

---

## 1. High‑Level Summary
This Bash driver creates per‑customer “traffic” CDR files for the **CrushSFTP** partner. It scans the daily mediation staging area (`$MNAASMainStagingDirDaily_v2`) for feed files matching patterns defined in a property file, filters and reshapes each record according to customer‑specific rules (including rating‑group and KYC look‑ups), writes the transformed rows to customer‑specific output directories, backs them up, normalises line endings, sets permissive file modes, generates checksum files, and updates a shared *process‑status* file. The script is scheduled by cron, protects against concurrent runs via a PID lock, and raises an SDP ticket + email on failure.

---

## 2. Key Functions & Responsibilities

| Function / Symbol | Responsibility |
|-------------------|----------------|
| **`traffic_file_creation`** | Core processing loop: iterates over feed sources, selects the newest matching file, creates customer directories, runs `awk` filters, merges KYC data (for rating‑group customers), writes output & backup files, normalises CR/LF, sets permissions, creates checksum (`cksum`) files, and logs progress. |
| **`terminateCron_successful_completion`** | Clears the “running” flag, records success status, timestamps, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step` (via caller) and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates the status file to *Failure*, checks whether an SDP ticket has already been raised, and if not sends a templated email to the SDP ticketing system (`insdp@tatacommunications.com`). |
| **Main script block** | PID‑lock handling, flag validation, invocation of `traffic_file_creation`, and final termination handling. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_create_customer_cdr_traffic_files_crushsftp.properties` – defines associative arrays used throughout: <br>• `FEED_FILE_PATTERN` (feed → filename glob) <br>• `MNAAS_Create_Customer_traffic_files_filter_condition` <br>• `MNAAS_Create_Customer_traffic_files_out_dir` / `*_backup_dir` <br>• `MNAAS_Create_Customer_traffic_files_column_position` <br>• `MNASS_Create_Customer_traffic_files_columns` (header line) <br>• `RATING_GROUP_CUSTOMER`, `MNAAS_Sub_Customer`, `RATING_GROUP_MAPPING` <br>• `MNAAS_Create_Customer_traffic_files_filter_condition_lookup` (KYC join) |
| **Environment / Global vars** | `MNAASMainStagingDirDaily_v2`, `MNAASMainStagingDirDaily_v1`, `MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename`, `MNAAS_Create_Customer_cdr_traffic_files_LogPath`, `MNAAS_Create_Customer_cdr_traffic_scriptname`, `SDP_ticket_from_email`, `SDP_ticket_cc_email`, `delimiter`, `sem_delimiter`, `MNAAS_Customer_temp_Dir` (temp work dir). |
| **File‑system inputs** | Feed files located under `$MNAASMainStagingDirDaily_v2/<pattern>` (e.g. `*.traffic`). |
| **File‑system outputs** | • Customer‑specific traffic files under `/backup1/MNAAS/Crushsftp/<customer>/…` <br>• Backup copies under `/backup1/MNAAS/Customer_CDR/Crushsftp/<customer>/…` <br>• Checksum files (`*.cksum` style) alongside each output/backup file <br>• Updated *process‑status* file (flags, PID, timestamps) <br>• Log file defined by `$MNAAS_Create_Customer_cdr_traffic_files_LogPath`. |
| **Side effects** | • Creation of many directories (if missing). <br>• Modification of file permissions to `777`. <br>• Removal of temporary `_TEMP` files after KYC join. <br>• Potential movement of source file to `$MNAASMainStagingDirDaily_v1` (commented out in current version). |
| **External services** | None directly invoked, but downstream processes may pick up the generated files via SFTP or internal file‑transfer pipelines. The SDP ticketing system is contacted via `mailx`. |

---

## 4. Integration Points & System Context

| Connected Component | How this script interacts |
|---------------------|---------------------------|
| **Other “traffic” scripts** (`MNAAS_create_customer_cdr_traffic_files_*.sh`) | Same staging area, same process‑status file; they are mutually exclusive via the PID lock. |
| **Mediation layer** | Supplies raw CDR traffic files into `$MNAASMainStagingDirDaily_v2`. |
| **KYC data repository** | Reads `/app/hadoop_users/MNAAS/MNAAS_Intermediatefiles/Daily/KYC_DataFiles/Files_withoutDups/KYC_All_Time_Snapshot.csv` for rating‑group customers. |
| **Rating‑group configuration** | Uses associative arrays (`RATING_GROUP_CUSTOMER`, `MNAAS_Sub_Customer`, `RATING_GROUP_MAPPING`) defined in the properties file. |
| **SDP ticketing/email system** | On failure, sends a templated email to `insdp@tatacommunications.com`. |
| **Downstream SFTP/FTP** | Generated files are later transferred to the CrushSFTP partner (outside this script). |
| **Process‑status monitor** | The shared status file (`*_ProcessStatusFilename`) is read/written by many scripts to coordinate runs and raise alerts. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID lock may be stale if script crashes. | Duplicate processing, file corruption. | Add a watchdog that clears stale PID after a configurable timeout; monitor the status file for long‑running flags. |
| **Missing or malformed property entries** (e.g., undefined associative array keys). | Script aborts with “unbound variable” or empty directories. | Validate required keys at start of `traffic_file_creation`; fail fast with clear log message. |
| **Disk‑space exhaustion** on output or backup directories. | Incomplete files, checksum mismatch. | Implement pre‑run free‑space check; alert if < X GB. |
| **Permission drift** – files set to `777` may violate security policy. | Exposure of sensitive CDR data. | Review required permissions; consider `660` with proper group ownership; make permission level configurable. |
| **KYC join failure** (missing KYC CSV or format change). | Output rows missing required fields. | Verify existence and schema of KYC file before processing; fallback to header‑only file with warning. |
| **Awk command injection** – variables not quoted when building commands. | Potential command execution errors. | Use proper quoting (`printf %q`) or switch to `awk -v var="$value"` style; add unit tests for edge cases. |
| **Checksum mismatch not detected** – script only creates checksum, never validates. | Corrupted files may be transferred. | Add optional verification step after file creation or integrate with downstream validation pipeline. |

---

## 6. Typical Execution & Debugging Workflow

1. **Scheduled run** – Cron invokes the script (e.g. `0 2 * * * /path/MNAAS_create_customer_cdr_traffic_files_crushsftp.sh`).  
2. **Lock check** – Script reads the PID from the process‑status file; if another instance is running it exits with a log entry.  
3. **Flag validation** – Reads `Create_Customer_traffic_files_flag`; proceeds only if `0` or `1`.  
4. **Processing** – `traffic_file_creation` runs; logs are written to `$MNAAS_Create_Customer_cdr_traffic_files_LogPath`.  
5. **Completion** – On success, `terminateCron_successful_completion` clears the flag and records a success timestamp. On error, `terminateCron_Unsuccessful_completion` triggers an SDP ticket.  

**Debugging steps**

| Step | Command / Action |
|------|-------------------|
| View latest log | `tail -f $MNAAS_Create_Customer_cdr_traffic_files_LogPath` |
| Force a single run (bypass cron) | `MNAAS_FlagValue=0 $HOME/MNAAS_create_customer_cdr_traffic_files_crushsftp.sh` (ensure proper env vars are exported). |
| Check PID lock | `cat $MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename | grep MNAAS_Script_Process_Id` |
| Verify property values | `grep -E 'FEED_FILE_PATTERN|MNAAS_Create_Customer_traffic_files_' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files_crushsftp.properties` |
| Simulate missing file | Move a feed file out of the staging dir and re‑run; observe “No … files” log entry. |
| Run with Bash `-x` (already set) and capture output | `bash -x MNAAS_create_customer_cdr_traffic_files_crushsftp.sh > /tmp/debug.out 2>&1` |

---

## 7. External Configuration & Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files_crushsftp.properties` | Holds all associative arrays and constants referenced throughout the script (feed patterns, column mappings, filter conditions, directory names, rating‑group data). |
| `MNAASMainStagingDirDaily_v2` | Source directory for raw mediation traffic files. |
| `MNAASMainStagingDirDaily_v1` | Archive directory for processed source files (currently commented out). |
| `MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename` | Shared status file tracking flags, PID, timestamps, email‑ticket flag. |
| `MNAAS_Create_Customer_cdr_traffic_files_LogPath` | Absolute path to the script’s log file. |
| `MNAAS_Create_Customer_cdr_traffic_scriptname` | Script name used in status file and log entries. |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` | Email addresses used when raising an SDP ticket. |
| `delimiter` (e.g., `;`) | Separator used when prefixing filenames with customer code. |
| `sem_delimiter` (e.g., `.cksum`) | Suffix added to checksum files. |
| `MNAAS_Customer_temp_Dir` | Temporary working directory for intermediate files. |
| `RATING_GROUP_CUSTOMER`, `MNAAS_Sub_Customer`, `RATING_GROUP_MAPPING` | Rating‑group specific configuration (defined in the properties file). |
| `MNAAS_Create_Customer_traffic_files_filter_condition_lookup` | Awk condition used to join KYC data. |

---

## 8. Suggested Improvements (TODO)

1. **Modularise the processing logic** – Extract the large `traffic_file_creation` body into smaller, testable functions (e.g., `prepare_customer_dirs`, `run_awk_filter`, `apply_kyc_join`, `finalise_file`). This will simplify maintenance and enable unit testing of each step.  
2. **Add robust error handling & validation** – Before the main loop, verify that all required associative arrays contain the expected keys for every feed/customer combination; abort early with a clear error if any are missing. Also, capture the exit status of each `awk`/`cksum` command and log failures rather than proceeding silently.  

---