**File:** `move-mediation-scripts\bin\MNAAS_move_files_from_staging.sh`  

---

## 1. High‑level purpose (one‑paragraph)

This script is the central orchestrator for the daily “move” stage of the MNAAS (Mobile Network Analytics & Assurance Suite) data‑pipeline. It validates that incoming CSV files in the staging area are unique (via a Java duplicate‑check utility), verifies checksum/record‑count consistency, moves each file set (traffic, actives, activations, tolling, SIM inventory, TAP errors, failed events, IMEI‑change) from the raw staging directory to a versioned “processed” directory, and mirrors the files to a remote real‑time server via SCP when running in production. For every step it updates a process‑status file, writes detailed logs, and raises SDP tickets / sends email alerts when files are missing, checksum mismatches occur, or the script itself fails.

---

## 2. Key functions / components

| Function | Responsibility |
|----------|----------------|
| **FilesDupsCheck** | Calls the Java class `com.tcl.mnaas.duplicatecheck.Dups_file_removal` to purge duplicate files from the staging area, updates status flags, and creates an SDP ticket + email on failure. |
| **generic_mv_files** | Core mover: for each raw file in a supplied directory, validates the accompanying `.sem` checksum file (row‑count & column‑count), moves the pair to the *good* directory (or reject directory on mismatch), optionally SCPs the file to the remote real‑time path, and logs the outcome. |
| **email_on_reject_triggering_step** | Sends a low‑priority SDP ticket / email when a file is rejected because its checksum does not match. |
| **mv_traffic_files**, **mv_actives_files**, **mv_activations_files**, **mv_tolling_files**, **mv_siminventory_files**, **mv_tap_error_files**, **mv_failed_events_files**, **mv_imeichange_files** | One function per file‑type group. Each: updates the process‑status flag, checks for file presence, raises a “missing‑file” alert if none appear for > 1 h (or 30 days for non‑move traffic), moves files via `generic_mv_files` (or simple `mv` for some types), and records the processing timestamp. |
| **terminateCron_successful_completion** | Writes a final “Success” status, timestamps, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Writes a “Failure” status, triggers a generic SDP ticket/email, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Generic failure notifier used by the script itself (not per‑file). |
| **email_and_SDP_ticket_triggering_step_validation** | Sends a formatted HTML email (and SDP ticket when the remote Windows/Telena job is not delivering files) to the Hadoop support team. |
| **Main driver block** | Prevents concurrent runs (PID lock), reads the current `MNAAS_Daily_ProcessStatusFlag` from the status file, and invokes the appropriate subset of the `mv_*` functions based on that flag. |

---

## 3. Inputs, outputs & side‑effects

| Category | Details |
|----------|---------|
| **Configuration / env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_move_files_from_staging.properties`. Expected variables (examples): <br>• `MNAASMainStagingDirDaily` – raw staging path<br>• `MNAASMainStagingDirDaily_v2` – processed “good” path<br>• `MNAASRejectedFilePath` – reject folder<br>• `MNAAS_move_files_from_staging_LogPath` – log file (date‑appended)<br>• `MNAAS_move_files_from_staging_ProcessStatusFileName` – key‑value status file<br>• `MNAAS_move_files_from_staging_ScriptName` – script identifier<br>• `ENV_MODE` – `PROD` or other (controls SCP)<br>• Remote SCP credentials (`SSH_MOVE_USER`, `SSH_MOVE_DAILY_SERVER`, `SSH_MOVE_DAILY_SERVER_PORT`, `SSH_MOVE_REALTIME_PATH`)<br>• Email lists (`MOVE_DEV_TEAM`, `MOVE_TELENA_WINDOWS_TO_EMAIL`, `MOVE_TELENA_WINDOWS_CC_EMAIL`, `SDP_ticket_from_email`, `SDP_ticket_cc_email`, etc.) |
| **External binaries / services** | Java runtime (for duplicate check), `scp` (remote copy), `mailx` / `sendmail` (email & SDP ticket generation), `logger` (syslog), `ps` (process lock), `uuidgen`, `hostname`. |
| **Input data** | CSV files matching patterns (e.g., `HOL_01_*TrafficDetails*.csv`, `SNG_01_*Actives*.csv`, `*IMEIChange*.csv`, etc.) located under `$MNAASMainStagingDirDaily`. Each CSV must have a companion `.sem` file containing the expected row count. |
| **Outputs** | - Moved files in `$MNAASMainStagingDirDaily_v2` (good) or `$MNAASRejectedFilePath` (reject). <br>- Optional remote copy to `$SSH_MOVE_REALTIME_PATH`. <br>- Log entries appended to `$MNAAS_move_files_from_staging_LogPath`. <br>- Updated key‑value pairs in `$MNAAS_move_files_from_staging_ProcessStatusFileName` (flags, timestamps, PID). <br>- SDP tickets / email alerts (via `mailx`). |
| **Assumptions** | • The properties file defines all required variables; missing variables cause script failure. <br>• The staging directory contains only files relevant to the current run (no stray files). <br>• The Java duplicate‑check JAR is present and compatible with the classpath. <br>• Network connectivity to the remote SCP server when `ENV_MODE=PROD`. <br>• Local filesystem permissions allow read/write/move for all involved directories. |

---

## 4. Interaction with other components

| Component | How this script connects |
|-----------|--------------------------|
| **MNAAS_Weekly_KYC_Feed_Loading.sh** (previous script) | Both scripts share the same status file (`$MNAAS_move_files_from_staging_ProcessStatusFileName`) and log path. The weekly KYC loader likely runs earlier in the day, populating the staging area with KYC‑related files that this script later moves. |
| **Duplicate‑check Java utility** | Invoked by `FilesDupsCheck` via `java -cp … $dups_file_removal_classname`. The utility reads the staging directory and a history folder to decide which files are duplicates. |
| **Remote Windows/Telena job** | The script monitors the arrival of Telena‑generated files (traffic, actives, etc.). If files are missing for > 1 h, it calls `email_and_SDP_ticket_triggering_step_validation` which sends an HTML notification to the Hadoop team and may raise an SDP ticket. |
| **Downstream ingestion jobs** | After files are moved to `$MNAASMainStagingDirDaily_v2`, downstream Hadoop/ETL jobs (not shown) pick them up for further processing (e.g., loading into Hive, Spark jobs). |
| **SDP ticketing system** | Emails with `%customer =Cloudera.Support@tatacommunications.com` are routed to the internal SDP ticketing platform, creating incidents for missing files or script failures. |
| **System logger** | All `logger -s` calls write to syslog, providing a central audit trail visible to operations. |

---

## 5. Operational risks & mitigations

| Risk | Impact | Recommended mitigation |
|------|--------|------------------------|
| **Stale PID lock** – script crashes leaving its PID in the status file, preventing future runs. | Daily pipeline stalls. | Add a sanity check on the PID’s start time (e.g., `ps -p $PID -o etime=`) and purge if older than a threshold. |
| **Missing or malformed `.sem` files** – leads to false rejects or silent skips. | Data loss / downstream errors. | Validate presence of `.sem` before processing; if missing, move file to a “manual‑review” folder and raise an alert. |
| **Duplicate‑check JAR incompatibility** – Java exit code non‑zero. | Entire day’s files may be considered duplicates and removed. | Version‑lock the JAR, run a quick sanity test at script start, and fallback to “skip duplicate check” with a warning if the class cannot be loaded. |
| **Network failure during SCP** – files not copied to remote real‑time path. | Real‑time consumers miss data. | Capture SCP exit code; on failure, retry a configurable number of times and log a distinct “SCP failure” alert. |
| **Email/SDP flood** – repeated missing‑file alerts generate many tickets. | Alert fatigue. | Use the “sent flag” (`MOVE_Delay_Email_*_sent`) already present; ensure it resets only after a successful file arrival. |
| **File‑system permission changes** – script cannot read/write directories. | Job aborts, no data moved. | Periodic health‑check cron that verifies read/write access to all configured paths and notifies ops. |
| **Hard‑coded path assumptions** – e.g., `/app/hadoop_users/...` may differ across environments. | Script fails in non‑standard deployments. | Externalize all paths into the properties file; add a validation step that aborts early if any required variable is empty. |

---

## 6. Typical execution / debugging workflow

1. **Run the script manually (as the Hadoop user):**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts/bin
   ./MNAAS_move_files_from_staging.sh
   ```
   - Ensure the properties file is readable.  
   - Observe console output (the script uses `logger -s` which also prints to stdout).  

2. **Check the PID lock:**  
   ```bash
   grep MNAAS_Script_Process_Id $MNAAS_move_files_from_staging_ProcessStatusFileName
   ps -p <pid> -o cmd=
   ```
   If the PID is stale, remove the line or kill the process.

3. **Inspect the status file:**  
   ```bash
   cat $MNAAS_move_files_from_staging_ProcessStatusFileName
   ```
   Verify `MNAAS_Daily_ProcessStatusFlag` and `MNAAS_job_status`.  

4. **Review logs:**  
   ```bash
   tail -f $MNAAS_move_files_from_staging_LogPath$(date +_%F)
   ```
   Look for “FilesDupsCheck IS SUCCESSFUL”, “Checksum issue identified”, or any “FAILED” messages.  

5. **Validate file movement:**  
   ```bash
   ls -l $MNAASMainStagingDirDaily_v2   # good files
   ls -l $MNAASRejectedFilePath         # rejects
   ```
   Confirm that the expected number of files were moved.  

6. **Debug a specific step:**  
   - Insert `set -x` at the top of the script (uncomment the existing line) to trace each command.  
   - Run the problematic function directly, e.g., `generic_mv_files "/path/to/traffic" "/good/path" "/reject/path" "22" "user@host:/dest"` to see detailed output.  

7. **Check email/SDP tickets:**  
   - Verify that the expected messages arrived in the support mailbox.  
   - Look for ticket IDs in the log lines “SDP ticket created & email generated”.  

---

## 7. External configuration & environment variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAASMainStagingDirDaily` | Source directory for raw daily files. |
| `MNAASMainStagingDirDaily_v2` | Destination directory for successfully processed files. |
| `MNAASRejectedFilePath` | Destination for files that fail checksum/column validation. |
| `MNAAS_move_files_from_staging_LogPath` | Base path for daily log files (date‑appended). |
| `MNAAS_move_files_from_staging_ProcessStatusFileName` | Central key‑value file tracking flags, timestamps, PID, etc. |
| `MNAAS_move_files_from_staging_ScriptName` | Human‑readable script identifier used in logs/emails. |
| `ENV_MODE` | Controls whether SCP is executed (`PROD` triggers remote copy). |
| `SSH_MOVE_USER`, `SSH_MOVE_DAILY_SERVER`, `SSH_MOVE_DAILY_SERVER_PORT`, `SSH_MOVE_REALTIME_PATH` | Remote SCP credentials and target path. |
| `MOVE_DEV_TEAM`, `MOVE_TELENA_WINDOWS_TO_EMAIL`, `MOVE_TELENA_WINDOWS_CC_EMAIL`, `SDP_ticket_from_email`, `SDP_ticket_cc_email` | Email distribution lists for alerts and SDP tickets. |
| `MNAAS_Telena_feed_windows_Job_name`, `MNAAS_Telena_feed_windows_server`, `MNAAS_Telena_feed_windows_user` | Metadata used in “missing‑file” notifications. |
| `dups_file_removal_classname`, `dups_history_of_files`, `DupsFilePath` | Parameters for the duplicate‑check Java class. |
| `Holland01_...`, `Singapore01_...` | Filename glob patterns for each file type (traffic, actives, etc.). |
| `SEM_EXTN` | Extension of the checksum file (e.g., `.sem`). |

All of the above are sourced via the `.properties` file at the top of the script.

---

## 8. Suggested improvements (TODO)

1. **Modularise the script into separate source files** – move each `mv_*` function into its own `.sh` module and source them. This reduces the monolithic size, eases unit testing, and allows independent versioning of file‑type handling logic.

2. **Add robust error handling for SCP** – wrap the `scp` command in a retry loop with exponential back‑off, capture its exit status, and write a distinct “SCP failure” entry to the status file and log. This prevents silent loss of the real‑time copy when network glitches occur.