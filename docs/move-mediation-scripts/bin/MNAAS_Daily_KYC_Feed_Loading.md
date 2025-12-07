**File:** `move-mediation-scripts\bin\MNAAS_Daily_KYC_Feed_Loading.sh`  

---

## 1. High‑Level Summary
This Bash script is the daily ingestion driver for the **KYC (Know‑Your‑Customer) feed** in the MNAAS mediation platform. It validates incoming usage files, removes duplicates, verifies SHA‑256 checksum files, moves clean files to the next staging area, archives checksum files, and routes corrupt or duplicate files to reject/duplicate folders. It also updates a shared *process‑status* file, sends operational e‑mail/SLA‑ticket notifications to the Windows Telena team when files are missing or delayed, and ensures only one instance runs at a time.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **FilesDupsCheck** | - Marks process start in the status file.<br>- Scans the daily KYC usage directory for files.<br>- Uses `$MNAASInterFilePath_Dups_Check` to detect duplicates.<br>- Moves duplicates to `$DupsFilePath`; logs non‑duplicates. |
| **mv_Daily_KYC_Feed_usage_files** | - Updates status flags.<br>- Detects *missing* daily usage files (24 h gap) → sends delay e‑mail & SDP ticket.<br>- When files appear after a delay, sends recovery e‑mail.<br>- For each file: verifies presence of a `.sha256` file, compares checksums, moves valid files to `$MNAASMainStagingDirDaily_afr_seq_check/` and archives the checksum; moves mismatched or missing‑checksum files to `$MNAASRejectedFilePath`.<br>- Updates last‑process timestamp and resets delay‑e‑mail flag. |
| **terminateCron_successful_completion** | - Writes *Success* status, clears flags, timestamps the run, logs completion, and exits 0. |
| **terminateCron_Unsuccessful_completion** | - Logs failure, writes *Failure* status, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | - Generates a low‑priority SDP ticket via `mailx` when the script fails (used by `terminateCron_Unsuccessful_completion`). |
| **email_and_SDP_ticket_triggering_step_validation** | - Sends a formatted HTML e‑mail to the Windows Telena team indicating whether files are being received.<br>- If files are not received, also creates an SDP ticket. |
| **Main Execution Block** | - Reads previous PID from the shared status file and ensures only one instance runs.<br>- Writes its own PID.<br>- Reads the process‑status flag (`MNAAS_Daily_ProcessStatusFlag`) and decides which functions to invoke (duplicate check + move, or move only).<br>- Handles unexpected flag values with a failure termination. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed.properties` – defines all `$MNAAS_*` variables (paths, filenames, e‑mail addresses, job names, etc.). |
| **Process‑status file** | `$MNAAS_Daily_KYC_Feed_ProcessStatusFileName` – a plain‑text key/value file that the script reads/writes to coordinate state, PID, timestamps, flags, and e‑mail‑sent markers. |
| **Input directories** | - `$MNAASMainStagingDirDaily/$Daily_KYC_Feed_Daily_Usage_ext` – raw daily KYC usage files.<br>- `$MNAASInterFilePath_Dups_Check` – a flat file containing names of already‑processed files for duplicate detection. |
| **Output / movement** | - Valid files → `$MNAASMainStagingDirDaily_afr_seq_check/` (next stage).<br>- Corresponding `.sha256` files → `$Daily_KYC_BackupDir/` (archive).<br>- Duplicate files → `$DupsFilePath`.<br>- Corrupt or checksum‑missing files → `$MNAASRejectedFilePath`. |
| **Logs** | All `stderr` and explicit `logger -s` messages are appended to `$MNAAS_Daily_KYC_Feed_LogPath`. |
| **External services** | - **Mail**: `mailx` and `/usr/sbin/sendmail` for e‑mail/SDP ticket generation.<br>- **Windows Telena job**: referenced only via e‑mail; no direct network call. |
| **Assumptions** | - All directories exist and are writable by the script user.<br>- `mailx`/`sendmail` are correctly configured and can reach internal ticketing mailboxes.<br>- The properties file supplies valid absolute paths and e‑mail addresses.<br>- No other process modifies the status file concurrently (PID lock is the only guard). |

---

## 4. Interaction with Other Scripts / Components  

| Component | Interaction |
|-----------|-------------|
| **Other MNAAS daily scripts** | Share the same *process‑status* file (`$MNAAS_Daily_KYC_Feed_ProcessStatusFileName`) to coordinate flags and PID. They may also append to `$MNAASInterFilePath_Dups_Check`. |
| **Duplicate‑check utility** | Not present in this file; the duplicate list (`$MNAASInterFilePath_Dups_Check`) is generated elsewhere (likely a prior ingestion step). |
| **Windows Telena ingestion job** | The script only notifies the Windows team via e‑mail when files are missing or delayed; the actual file transfer is performed by the Windows job. |
| **Downstream Hadoop jobs** | Files moved to `$MNAASMainStagingDirDaily_afr_seq_check/` are expected to be consumed by subsequent Hadoop/MapReduce or Spark jobs (outside the scope of this script). |
| **Ticketing system** | SDP tickets are created by sending specially‑formatted e‑mail to `insdp@tatacommunications.com`. The ticketing backend parses the `%`‑prefixed fields. |
| **Cron scheduler** | The script is intended to be invoked by a daily cron entry (not shown). The PID guard prevents overlapping runs. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **PID lock race condition** – if the script crashes, the stale PID remains, blocking future runs. | Stalled daily ingestion. | Implement a lock file with timeout (e.g., `flock`) and clean up stale locks on start. |
| **In‑place `sed` on the status file** – concurrent writes could corrupt the file. | Loss of state, false alerts. | Serialize access (e.g., `flock`) or switch to atomic write (write to temp then `mv`). |
| **Missing or malformed checksum files** – script only logs and skips processing, potentially leaving files stranded. | Data loss or backlog. | Add a configurable retry/alert for missing checksum files; move such files to a “needs‑review” folder. |
| **Hard‑coded paths & magic numbers** (e.g., 86400 seconds). | Maintenance difficulty, environment drift. | Externalize thresholds and paths to the properties file. |
| **No error handling on `mv` commands** – failures (e.g., permission) are not captured. | Silent data loss. | Check exit status of `mv` and log/alert on failure. |
| **Mail delivery failures** – `mailx`/`sendmail` may silently drop messages. | Undetected operational issues. | Verify mail command exit status; optionally retry or write to a “mail‑failed” queue. |
| **Duplicate detection based on a flat file** – may grow large and become a performance bottleneck. | Slow scans, missed duplicates. | Rotate/archieve the duplicate list periodically; consider using a database or indexed file. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed.properties` is present and exported variables are correct.  
   - Verify write permissions on all directories referenced in the properties file.  
   - Confirm `mailx` and `sendmail` are functional (test with a simple mail).  

2. **Manual Execution**  
   ```bash
   # Optional: enable tracing for detailed output
   set -x
   ./MNAAS_Daily_KYC_Feed_Loading.sh
   ```
   - The script will automatically append errors to the log defined by `$MNAAS_Daily_KYC_Feed_LogPath`.  
   - Use `tail -f $MNAAS_Daily_KYC_Feed_LogPath` to monitor progress.  

3. **Debugging Tips**  
   - **Check PID lock**: `cat $MNAAS_Daily_KYC_Feed_ProcessStatusFileName | grep MNAAS_Script_Process_Id` to see the stored PID.  
   - **Validate duplicate list**: `grep -i <filename> $MNAASInterFilePath_Dups_Check`.  
   - **Force a checksum mismatch**: modify a `.sha256` file and re‑run to verify reject handling.  
   - **Simulate missing files**: empty the staging directory and run; confirm the 24‑hour delay logic (you may temporarily set `diff_time_last_process` threshold to a lower value for testing).  

4. **Exit Codes**  
   - `0` – successful completion (status file set to *Success*).  
   - `1` – abnormal termination (status file set to *Failure* and an SDP ticket is raised).  

---

## 7. External Configuration & Environment Variables  

| Variable (populated from properties) | Purpose |
|--------------------------------------|---------|
| `MNAAS_Daily_KYC_Feed_LogPath` | Path to the script’s log file (stderr redirection). |
| `MNAAS_Daily_KYC_Feed_ProcessStatusFileName` | Shared status/flag file used for coordination and PID tracking. |
| `MNAASMainStagingDirDaily` | Base directory for daily KYC staging files. |
| `Daily_KYC_Feed_Daily_Usage_ext` | Sub‑directory (or file pattern) containing the raw usage files. |
| `MNAASInterFilePath_Dups_Check` | File that lists already‑processed filenames for duplicate detection. |
| `MNAAS_Telena_feed_windows_Job_name`, `MNAAS_Telena_feed_windows_server`, `MNAAS_Telena_feed_windows_user` | Metadata used in notification e‑mails to the Windows team. |
| `MOVE_TELENA_WINDOWS_TO_EMAIL`, `MOVE_TELENA_WINDOWS_CC_EMAIL` | Recipient lists for delay/recovery notifications. |
| `MNAAS_support_team_email` | CC address for SDP tickets. |
| `MNAASRejectedFilePath`, `MNAASMainStagingDirDaily_afr_seq_check`, `Daily_KYC_BackupDir` | Destination directories for rejected, next‑stage, and checksum‑archive files. |
| `SDP_ticket_from_email` | “From” address used when creating SDP tickets. |
| `MOVE_DEV_TEAM` | CC address for failure tickets. |
| `MNAAS_Daily_KYC_Feed_ScriptName` | Human‑readable script identifier (used in logs/tickets). |

*All of the above are expected to be defined in the sourced `.properties` file; the script does not provide defaults.*

---

## 8. Suggested Improvements (TODO)

1. **Replace PID file lock with `flock`** – ensures atomic lock acquisition and automatic release on process exit, eliminating stale‑PID issues.  
2. **Add robust error handling for file operations** – capture exit codes of `mv`, `sed`, and `sha256sum`; on failure, write to a dedicated “operation‑error” log and optionally raise an alert.  

---