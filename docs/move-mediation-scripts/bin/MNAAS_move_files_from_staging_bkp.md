**MNAAS_move_files_from_staging_bkp.sh – High‑Level Documentation**  

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_move_files_from_staging_bkp.sh` is the central “move” driver that processes daily mediation files arriving in the Hadoop staging area. It first runs a Java duplicate‑check job, then iterates through a predefined list of file‑type patterns (traffic, actives, activations, tolling, SIM inventory, TAP errors, failed events, IMEI‑change) for the two sites *Holland* (HOL) and *Singapore* (SNG). For each type it validates the presence of a matching checksum (`*.sem`) file, verifies record‑count consistency, moves the file to the versioned directory (`…/MNAAS_DailyFiles_v1/`), optionally SCPs it to a downstream Windows/Telena server, and updates a shared process‑status property file. Missing or malformed files trigger email alerts and automatic SDP‑ticket creation. The script is guarded against concurrent execution via a PID lock stored in the same status file.  

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **FilesDupsCheck** | Executes the Java duplicate‑removal class (`com.tcl.mnaas.duplicatecheck.Dups_file_removal`). Updates status flags, logs success/failure, and creates an SDP ticket + email on failure. |
| **generic_mv_files** | Core mover used by traffic‑file handling. For each raw file it checks the companion `.sem` file, validates row‑count and column‑count consistency, moves the pair to the *good* directory or to the *reject* directory, and optionally SCPs to the Windows server (only in PROD). |
| **email_on_reject_triggering_step** | Sends a low‑priority SDP email when a file is rejected because the checksum/column count does not match. |
| **mv_traffic_files** | Orchestrates processing of HOL and SNG traffic files (`*_Traffic_Details*`). Handles missing‑file detection, delay‑email throttling, and calls `generic_mv_files`. |
| **mv_actives_files** | Same pattern as traffic but for *actives* files (`*_Actives*`). |
| **mv_activations_files** | Same pattern as traffic but for *activations* files (`*_Activations*`). |
| **mv_tolling_files** | Same pattern as traffic but for *tolling* files (`*_Tolling*`). |
| **mv_siminventory_files** | Same pattern as traffic but for *SIM inventory* files (`*_SimInventory*`). |
| **mv_tap_error_files** | Simple move for TAP‑error files (no alert logic). |
| **mv_failed_events_files** | Simple move for failed‑event files (no alert logic). |
| **mv_imeichange_files** | Moves IMEI‑change files for both sites; logs missing‑file warnings. |
| **terminateCron_successful_completion** | Writes final “Success” flags to the status file, logs end‑time, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers a generic SDP ticket/email, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Generic failure‑notification routine used by `terminateCron_Unsuccessful_completion`. |
| **email_and_SDP_ticket_triggering_step_validation** | Sends a detailed “window‑job not receiving” or “receiving” email (HTML) and creates an SDP ticket when a file‑type is missing for > 1 hour. |
| **process‑lock block (bottom of script)** | Checks the PID stored in the status file; if another instance is running the script aborts gracefully. |

---

### 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_move_files_from_staging.properties` – defines all directory paths, file‑type extensions, log file locations, email lists, Windows‑job parameters, SCP ports, etc. |
| **Environment variables** | `ENV_MODE` (used to gate SCP), `MNAAS_setparameter` (presumably sets `set -e`/`set -x`), `MNAAS_FlagValue` (read from status file), plus any variables referenced in the property file (e.g., `MNAASMainStagingDirDaily`, `MNAASMainStagingDirDaily_v2`, `MNAASRejectedFilePath`, `MNAAS_move_files_from_staging_LogPath`, `MNAAS_move_files_from_staging_ProcessStatusFileName`, `MNAAS_move_files_from_staging_ScriptName`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, etc.). |
| **External services** | • Java runtime (`java -cp …`) – duplicate‑check JAR.<br>• Local file system (staging, good, reject directories).<br>• `logger` (syslog).<br>• `mailx` / `sendmail` for SDP tickets and alerts.<br>• Optional `scp` to a Windows/Telena server (port defined in properties). |
| **Primary inputs** | Files matching the patterns defined in the property file (e.g., `HOL_01_*TrafficDetails*.csv`, `SNG_01_*Actives*.csv`, etc.) and their companion `.sem` checksum files. |
| **Primary outputs** | • Moved files under `${MNAASMainStagingDirDaily_v2}` (good) or `${MNAASRejectedFilePath}` (reject).<br>• Updated process‑status property file (flags, timestamps, PID).<br>• Log entries appended to `${MNAAS_move_files_from_staging_LogPath}`.<br>• Emails/SDP tickets on errors or missing‑file alerts. |
| **Assumptions** | • The property file exists and contains all required variables.<br>• The staging directory is exclusively used by this pipeline (no external writes while the script runs).<br>• `mailx` and `sendmail` are correctly configured for outbound mail.<br>• The Java duplicate‑check class is compatible with the current Hadoop/Java version.<br>• The script runs under a user with read/write/execute permissions on all referenced paths and the ability to run `scp`. |

---

### 4. Interaction with Other Components  

| Component | How this script connects |
|-----------|--------------------------|
| **Scheduler (cron)** | Typically invoked by a daily cron entry; the PID lock prevents overlapping runs. |
| **MNAAS_*_process scripts** | Those scripts read the same *process‑status* property file to decide whether to start downstream jobs (e.g., loading into Hive, feeding downstream systems). |
| **Java duplicate‑check JAR** | Called from `FilesDupsCheck`; the JAR updates a “duplicate‑file removal” history directory. |
| **Windows/Telena ingestion service** | When `ENV_MODE=PROD`, `generic_mv_files` SCPs the good files to `${SSH_MOVE_DAILY_SERVER}`; the Windows side is expected to pick them up via a scheduled job (`$MNAAS_Telena_feed_windows_Job_name`). |
| **SDP ticketing system** | Emails are sent to `insdp@tatacommunications.com` (or other addresses) with a specific format that the SDP system parses to create incidents. |
| **Logging/monitoring** | All `logger` calls go to syslog; the log file path is also defined in the property file for per‑run archival. |
| **Other move scripts** | The “generic_mv_files” function is reused by traffic handling; similar patterns exist in other move scripts (e.g., `MNAAS_move_files_from_staging_original.sh`). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID lock may be stale if the script crashes. | Duplicate processing, file loss, inconsistent status flags. | Implement a lock‑file with a timeout; on start, verify the PID is still alive (`ps -p`). |
| **Java duplicate‑check failure** – non‑zero exit leads to SDP ticket but processing continues. | Potential duplicate records downstream. | Add retry logic or fallback to a “skip‑duplicate‑check” mode after a configurable number of attempts. |
| **Missing `.sem` files** – leads to reject and email flood. | Alert fatigue, mailbox overload. | Introduce a configurable “max‑reject‑per‑run” threshold and aggregate alerts. |
| **Hard‑coded mail recipients** – changes require script edit. | Missed notifications after personnel changes. | Externalize recipient lists to the properties file. |
| **SCP to Windows server** – disabled in non‑PROD but may be forgotten in PROD. | Files not delivered downstream. | Add a health‑check step that verifies connectivity before processing. |
| **Large directory listings (`ls -1 $dir`)** – may cause performance issues when many files exist. | Slow runs, possible timeouts. | Use `find -maxdepth 1 -type f` with pattern matching, or process files in batches. |
| **Use of `sed -i` on the same status file from multiple functions** – race condition if functions are parallelized in the future. | Corrupted status file. | Serialize all status‑file writes or use a temporary file + atomic rename. |
| **Unvalidated environment variables** – missing vars cause script to abort silently. | Unexpected failures. | Add an early “validate_config” block that checks required variables and exits with a clear message. |

---

### 6. Typical Run / Debug Procedure  

1. **Preparation**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_move_files_from_staging.properties` is present and readable.  
   - Verify that the staging directory (`$MNAASMainStagingDirDaily`) contains the expected inbound files.  
   - Confirm that the user has write permission on the good (`*_v2`) and reject directories, the log file, and the status file.  

2. **Dry‑run (no side‑effects)**  
   ```bash
   export ENV_MODE=DEV   # disables SCP
   set -x                 # enable Bash tracing
   ./MNAAS_move_files_from_staging_bkp.sh
   ```
   - Observe the trace output; look for any “command not found” or variable expansion issues.  

3. **Check PID lock**  
   ```bash
   grep MNAAS_Script_Process_Id $MNAAS_move_files_from_staging_ProcessStatusFileName
   ps -p <pid>
   ```
   - If a stale PID is present, manually clear the entry or kill the orphaned process.  

4. **Validate status flags**  
   ```bash
   grep MNAAS_Daily_ProcessStatusFlag $MNAAS_move_files_from_staging_ProcessStatusFileName
   ```
   - The flag should be `0` (idle) or `1` (ready for duplicate check).  

5. **Run in production**  
   ```bash
   ./MNAAS_move_files_from_staging_bkp.sh
   ```
   - Monitor the log file (`tail -f $MNAAS_move_files_from_staging_LogPath`) and syslog for `logger` entries.  

6. **Post‑run verification**  
   - Confirm that files have been moved to `${MNAASMainStagingDirDaily_v2}` and that any rejected files appear under `${MNAASRejectedFilePath}`.  
   - Check that the status file now shows `MNAAS_Daily_ProcessStatusFlag=0` and `MNAAS_job_status=Success`.  
   - Verify that any expected alert emails were sent (mail queue, inbox).  

7. **Troubleshooting**  
   - Non‑zero exit from the Java duplicate‑check: inspect `$MNAAS_move_files_from_staging_LogPath` for stack traces.  
   - Missing `.sem` files: ensure upstream processes generate them; look for naming mismatches.  
   - SCP failures (PROD only): test connectivity manually (`scp -P $SSH_MOVE_DAILY_SERVER_PORT testfile $SSH_MOVE_USER@$SSH_MOVE_DAILY_SERVER:/tmp/`).  

---

### 7. External Config / Environment Dependencies  

| Item | Description | Usage |
|------|-------------|-------|
| **MNAAS_move_files_from_staging.properties** | Central property file; defines directories, file extensions, log paths, email lists, Windows job details, SCP credentials, etc. | Sourced at script start (`.`). |
| **ENV_MODE** | Determines whether SCP is executed (`PROD` enables remote copy). | Checked inside `generic_mv_files`. |
| **$setparameter** | Likely expands to `set -e` or `set -x`; controls Bash error handling / tracing. | Executed early (`$setparameter`). |
| **Java classpath & JAR** (`$CLASSPATHVAR`, `$MNAAS_Main_JarPath`) | Required for duplicate‑check execution. | Passed to `java -cp …`. |
| **Mail configuration** (`mailx`, `sendmail`, `$SDP_ticket_from_email`, `$MOVE_DEV_TEAM`, `$MOVE_TELENA_WINDOWS_TO_EMAIL`, etc.) | Used for alert and SDP ticket generation. | Embedded in email functions. |
| **SSH variables** (`$SSH_MOVE_DAILY_SERVER`, `$SSH_MOVE_USER`, `$SSH_MOVE_DAILY_SERVER_PORT`, `$SSH_MOVE_REALTIME_PATH`) | Destination for remote copy of good files. | Used in `generic_mv_files`. |
| **Process‑status file** (`$MNAAS_move_files_from_staging_ProcessStatusFileName`) | Shared state across the pipeline; holds flags, timestamps, PID, email‑sent markers. | Read/written throughout the script. |

---

### 8. Suggested Improvements (TODO)  

1. **Add a configuration validation block** – early in the script iterate over a list of required variables (directories, email addresses, ports) and abort with a clear error if any are missing or unreadable. This prevents silent failures caused by a malformed property file.  

2. **Replace the `ls | wc -l` pattern with `find`** – using `find "$dir" -maxdepth 1 -type f -name "$pattern" -print0 | wc -l` (or `-printf '.' | wc -c`) eliminates issues with filenames containing spaces or newlines and scales better when the staging directory holds many files.  

--- 

*End of documentation.*