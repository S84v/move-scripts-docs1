**File:** `move-mediation-scripts/bin/MNAAS_move_files_from_staging_ipvprobe.sh`

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily ingestion of IPVProbe usage files from the Hadoop staging area. It first checks for duplicate file arrivals, moves valid files to a version‑1 staging directory after verifying MD5 checksums against accompanying `.sem` files, and generates alerts (email & SDP ticket) when files are missing, duplicated, or fail checksum validation. The script maintains a process‑status file to coordinate concurrent runs, logs all actions, and updates a job‑status flag for downstream loading jobs.

---

## 2. Key Functions & Responsibilities

| Function | Purpose |
|----------|---------|
| **FilesDupsCheck** | Scans the daily staging directory for files whose names appear in the duplicate‑check list (`$MNAASInterFilePath_Dups_Check`). Duplicates are moved to a dedicated “duplicates” folder and an incident email is sent. |
| **mv_ipvprobe_usage_files** | Handles three scenarios: <br>1. No files present → checks elapsed time since last successful run and, if > 24 h, sends a “missing file” alert. <br>2. Files present → validates checksum using the `.sem` file; on success moves the file to `$MNAASMainStagingDirDaily_v1/`, otherwise moves both file and `.sem` to the reject folder and logs the event. <br>3. Updates the process‑status file with the latest processing timestamp and resets the “delay‑email‑sent” flag. |
| **terminateCron_successful_completion** | Writes a *Success* status to the process‑status file, logs completion timestamps, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs a failure, writes a *Failure* status, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Generates a generic SDP ticket (via `mailx`) when the script fails and ensures the ticket is created only once per run. |
| **email_and_SDP_ticket_triggering_step_validation** | Sends a detailed notification to the Windows‑team (and optionally creates an SDP ticket) indicating whether the Telena‑to‑Hadoop file feed is currently receiving data. Used for both “file missing” and “file resumed” scenarios. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | `MNAAS_move_files_from_staging_ipvprobe.properties` (sourced). Contains paths, filenames, email addresses, job names, and flag defaults. |
| **External Files / Directories** | - `$MNAASMainStagingDirDaily/$IPVPROBE_Daily_Usage_ext` – raw daily usage files (e.g., `*.txt`). <br>- `$MNAASInterFilePath_Dups_Check` – list of known duplicate filenames. <br>- `$DupsFilePath` – archive for duplicate files. <br>- `$MNAASMainStagingDirDaily_v1/` – destination for validated files. <br>- `$MNAASRejectedFilePath` – reject folder for checksum failures. <br>- `$MNAAS_move_files_from_staging_ipvprobe_ProcessStatusFileName` – key‑value status file used for coordination and flag storage. |
| **Logs** | `$MNAAS_move_files_from_staging_ipvprobe_LogPath` – appended with all `logger` output and script‑level messages. |
| **Side Effects** | - Moves/renames files across staging, duplicate, and reject directories. <br>- Sends email alerts via `mailx` and raw SMTP (`sendmail`). <br>- Updates the process‑status file (flags, timestamps, PID). <br>- Writes entries to the system logger (`logger -s`). |
| **Assumptions** | - The properties file defines all required variables; missing definitions will cause runtime errors. <br>- The staging directory is exclusively used by this script (no external processes moving files concurrently). <br>- `mailx` and `sendmail` are correctly configured on the host. <br>- The `.sem` file naming convention (`<basename>.sem`) and MD5 checksum format are consistent. |

---

## 4. Interaction with Other Components

| Component | Relationship |
|-----------|--------------|
| **Up‑stream producers** | Other mediation jobs (e.g., `MNAAS_ipvprobe_tbl_Load.sh`) deposit raw IPVProbe usage files into `$MNAASMainStagingDirDaily`. |
| **Down‑stream consumers** | Subsequent loading scripts (e.g., `MNAAS_ipvprobe_daily_recon_loading.sh`) read from `$MNAASMainStagingDirDaily_v1/`. |
| **Process‑status file** | Shared with sibling scripts (e.g., `MNAAS_move_files_from_staging_bkp.sh`) to coordinate run flags and PID tracking. |
| **Duplicate‑check list** | Populated by a separate quality‑control job that identifies duplicate arrivals. |
| **Alerting / Ticketing** | Uses internal email distribution lists (`$MOVE_DEV_TEAM`, `$MNAAS_support_team_email`) and the SDP ticketing system via specially formatted `mailx` messages. |
| **Cron scheduler** | Typically invoked from a daily cron entry; the script self‑guards against overlapping runs using the PID stored in the status file. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale PID in status file** – if the script crashes, the PID remains and prevents future runs. | Production stop; files pile up. | Add a sanity check on PID age (e.g., `ps -p $PID -o etime=`) and clear it if older than a threshold. |
| **Missing or malformed `.sem` file** – leads to file being ignored and potentially lost. | Data loss / delayed processing. | Implement a fallback to move the raw file to a “manual‑review” folder and generate a high‑priority alert. |
| **Checksum mismatch** – may be caused by transient transfer errors. | Valid data rejected. | Add a retry mechanism (e.g., re‑calculate checksum after a short wait) before moving to reject. |
| **Email / SDP ticket failure** – alerts not sent. | Undetected incidents. | Capture exit status of `mailx`/`sendmail` and retry; also write to a persistent “alert‑failure” log for later review. |
| **Duplicate‑check list out‑of‑date** – false positives/negatives. | Unnecessary file moves or missed duplicates. | Schedule regular refresh of `$MNAASInterFilePath_Dups_Check` from the upstream deduplication process. |
| **Hard‑coded paths** – lack of portability. | Breakage on environment changes. | Move all path definitions into the properties file (already done) and enforce validation at script start. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_move_files_from_staging_ipvprobe.properties` is present and all variables are defined.  
   - Verify write permissions on the log, status, duplicate, reject, and destination directories.  
   - Confirm `mailx` and `sendmail` are functional (test with a simple mail command).

2. **Manual Execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/
   ./move-mediation-scripts/bin/MNAAS_move_files_from_staging_ipvprobe.sh
   ```
   - The script will append to the log file defined by `$MNAAS_move_files_from_staging_ipvprobe_LogPath`.  
   - Use `tail -f <log>` to watch progress.

3. **Debug Mode**  
   - Uncomment the line `#set -x` near the top to enable Bash trace output (adds each command to the log).  
   - Optionally export `BASH_DEBUG=1` before running to keep the trace.

4. **Checking Process Status**  
   ```bash
   grep MNAAS_Script_Process_Id $MNAAS_move_files_from_staging_ipvprobe_ProcessStatusFileName
   ps -p <PID>
   ```
   - If the PID is stale, manually clear the entry in the status file.

5. **Simulating Scenarios**  
   - **Duplicate file**: Add a filename that exists in `$MNAASInterFilePath_Dups_Check` to the staging dir and run the script; verify it moves to `$DupsFilePath` and an email is sent.  
   - **Missing checksum**: Drop a file without a matching `.sem`; the script should log “checksum file not found” and skip processing.  
   - **Checksum mismatch**: Corrupt a file after generating its `.sem`; the script should move both to the reject folder.

6. **Log Review**  
   - Success: Look for “Script … terminated successfully” and `MNAAS_job_status=Success`.  
   - Failure: Look for “terminated Unsuccessfully” and any “SDP ticket created” entries.

---

## 7. External Config / Environment Variables

| Variable (from properties) | Role |
|----------------------------|------|
| `MNAAS_move_files_from_staging_ipvprobe_LogPath` | Full path to the script’s log file. |
| `MNAAS_move_files_from_staging_ipvprobe_ProcessStatusFileName` | Central status/flag file shared across mediation jobs. |
| `MNAASMainStagingDirDaily` | Root staging directory for daily raw files. |
| `IPVPROBE_Daily_Usage_ext` | Filename extension/pattern for IPVProbe usage files (e.g., `*.txt`). |
| `MNAASInterFilePath_Dups_Check` | Path to the duplicate‑file list. |
| `MNAASMainStagingDirDaily_v1` | Destination for validated files. |
| `MNAASRejectedFilePath` | Folder for files failing checksum validation. |
| `DupsFilePath` | Archive location for duplicate files. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `MNAAS_support_team_email` | Email addresses used for alerts and ticket creation. |
| `MNAAS_Telena_feed_windows_Job_name`, `MNAAS_Telena_feed_windows_server`, `MNAAS_Telena_feed_windows_user` | Metadata for the Windows‑based Telena feed referenced in alert emails. |
| `MOVE_TELENA_WINDOWS_TO_EMAIL`, `MOVE_TELENA_WINDOWS_CC_EMAIL` | Recipients of the “feed status” notification. |
| `MNAAS_move_files_from_staging_ipvprobe_ScriptName` | Human‑readable script identifier used in logs and tickets. |

---

## 8. Suggested Improvements (TODO)

1. **PID Staleness Guard** – Add logic to detect and clear a PID that belongs to a non‑existent process (e.g., `if ! ps -p $PID > /dev/null; then clear PID`).  
2. **Centralised Alert Wrapper** – Consolidate the two email/SDP functions into a single reusable library (e.g., `MNAAS_alert.sh`) to avoid duplication and simplify future changes to ticket formatting or recipient lists.  

---