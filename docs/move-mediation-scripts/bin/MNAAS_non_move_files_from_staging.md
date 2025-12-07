**File:** `move-mediation-scripts/bin/MNAAS_non_move_files_from_staging.sh`

---

## 1. High‑level Summary
This Bash driver orchestrates the daily “move” phase of the MNAAS mediation pipeline. It scans the staging directory for a predefined set of Holland (HOL) and Singapore (SNG) feed files, moves each file to a version‑ed archive directory, updates a shared *process‑status* file, logs activity, and notifies downstream Windows‑based Telena ingestion jobs via email/SDP tickets when files are missing or when a Windows job stops receiving data. Execution is gated by a flag stored in the status file so that the script can resume from the appropriate step if a previous run was interrupted.

---

## 2. Key Functions & Responsibilities  

| Function | Purpose |
|----------|---------|
| **mv_traffic_files** | Move HOL & SNG *traffic* CSVs, copy to remote Windows server (scp) when `ENV_MODE=PROD`, update timestamps, handle missing‑file alerts. |
| **mv_actives_files** | Move *actives* CSVs, update status, generate missing‑file alerts. |
| **mv_activations_files** | Move *activations* CSVs, update status, generate missing‑file alerts. |
| **mv_tolling_files** | Move *tolling* CSVs, update status, generate missing‑file alerts (includes a typo‑bug in status‑file name – see Risks). |
| **mv_siminventory_files** | Move *SIM inventory* CSVs, update status, generate missing‑file alerts. |
| **mv_tap_error_files** | Move SNG *tap‑error* files; only logs missing files (no alert). |
| **mv_failed_events_files** | Move SNG *failed‑events* files; only logs missing files (no alert). |
| **mv_imeichange_files** | Move IMEI‑change CSVs for both regions; logs missing files, no alert. |
| **terminateCron_successful_completion** | Write final success flags to the status file, log end‑time, exit 0. |
| **terminateCron_Unsuccessful_completion** | Log failure, trigger a generic SDP ticket, exit 1. |
| **email_and_SDP_ticket_triggering_step** | Create a failure SDP ticket (via `mailx`) and mark the ticket‑created flag in the status file. |
| **email_and_SDP_ticket_triggering_step_validation** | Send a formatted notification to the Windows team (and optionally an SDP ticket) when a feed is missing or a Windows job stops receiving data. |
| **Main driver block** | Checks for an existing PID lock, reads the current `MNAAS_Daily_ProcessStatusFlag` from the status file and invokes the appropriate subset of the `mv_*` functions, then terminates. |

---

## 3. Inputs, Outputs & Side‑effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_move_files_from_staging.properties` – defines all variables used throughout the script (paths, file name patterns, SSH credentials, email addresses, log & status file locations, `ENV_MODE`, etc.). |
| **Environment variables** | Expected to be exported by the properties file or the calling environment: `MNAASMainStagingDirDaily`, `MNAASMainStagingDirDaily_v2`, `MNAAS_move_files_from_staging_LogPath`, `MNAAS_move_files_from_staging_ProcessStatusFileName`, `MNAAS_move_files_from_staging_ScriptName`, `SSH_MOVE_DAILY_SERVER`, `SSH_MOVE_USER`, `SSH_MOVE_REALTIME_PATH`, `SSH_MOVE_DAILY_SERVER_PORT`, `MOVE_TELENA_WINDOWS_TO_EMAIL`, `MOVE_TELENA_WINDOWS_CC_EMAIL`, `MNAAS_Telena_feed_windows_Job_name`, `MNAAS_Telena_feed_windows_server`, `MNAAS_Telena_feed_windows_user`, `MNAAS_support_team_email`, `SDP_ticket_from_email`, `SDP_ticket_cc_email`, `MOVE_DEV_TEAM`, etc. |
| **File system inputs** | Staging directory `$MNAASMainStagingDirDaily` containing files matching the patterns defined in the script (e.g. `HOL_01_*TrafficDetails*.csv`). |
| **File system outputs** | Archive directory `$MNAASMainStagingDirDaily_v2` where processed files are moved. Updated status file (`$MNAAS_move_files_from_staging_ProcessStatusFileName`). Log file (`$MNAAS_move_files_from_staging_LogPath`). |
| **External services** | - **SCP** to a Windows server (only in PROD). <br> - **mailx / sendmail** for email notifications and SDP ticket creation. <br> - **logger** (syslog) for audit trail. |
| **Side‑effects** | - Modifies the status file (multiple `sed -i` operations). <br> - Sends emails / creates SDP tickets. <br> - May trigger downstream Windows jobs via file arrival. |
| **Assumptions** | - All directories exist and are writable. <br> - `logger`, `mailx`, `sendmail`, `scp`, `uuidgen` are present in `$PATH`. <br> - The properties file correctly defines every variable referenced. <br> - The process‑status file is a simple `key=value` flat file. <br> - No concurrent runs (PID lock works). |

---

## 4. Interaction with Other Components  

| Component | Connection Point |
|-----------|------------------|
| **MNAAS_move_files_from_staging.properties** | Sourced at the top; provides all configurable values. |
| **MNAAS_move_files_from_staging_bkp.sh** (and similar “backup” scripts) | Share the same status file and log path; may be invoked by a higher‑level orchestrator after this script finishes. |
| **Windows Telena ingestion jobs** | Receive files via SCP; their health is monitored via the `email_and_SDP_ticket_triggering_step_validation` function. |
| **SDP ticketing system** | Triggered via `mailx` with a special `%customer` header; other scripts in the suite use the same pattern. |
| **Cron scheduler** | Typically executed as a daily cron job; the script checks for an existing PID to avoid overlapping runs. |
| **Other “move” scripts** (e.g. `MNAAS_move_files_from_staging_ipvprobe.sh`) | Operate on different data domains but use the same status‑file mechanism; the flag values (0‑8) are coordinated across the suite. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **PID lock race** – stale PID may block future runs. | Job stops processing for days. | Add a sanity check on PID age (`ps -p $PID -o etime=`) and clear if older than a threshold. |
| **`sed -i` on status file** – concurrent edits can corrupt the file. | Loss of state, false alerts. | Use `flock` around all status‑file updates or switch to a transactional format (e.g. SQLite). |
| **Hard‑coded file patterns** – missing or renamed feed files cause false “missing‑file” alerts. | Alert fatigue, missed real failures. | Centralise patterns in the properties file and validate existence before processing. |
| **SCP failures not checked** – network outage leaves files un‑copied but still moved locally. | Downstream Windows jobs miss data. | Capture SCP exit code; on failure, rollback the local `mv` or retry before moving. |
| **Typo in status‑file variable name** (`$MNAAS_move_files_from_staging_ProcessSMOVE_Delay_Email_tolling_HOL_senttatusFileName`). | Tolling alerts may never be cleared, causing repeated tickets. | Correct the variable name; add unit tests for each function. |
| **Missing environment variables** – script aborts with “unbound variable”. | Job failure, no processing. | Add a validation block after sourcing the properties file that checks required vars and exits with a clear message. |
| **Email/SDP flood** – repeated missing‑file alerts if a feed is genuinely delayed. | Spam, ticket backlog. | Implement a back‑off counter (e.g., only alert once per 6‑hour window). |
| **No error handling for `mv`** – if `mv` fails (e.g., permission), script continues. | Files remain in staging, later steps may re‑process. | Test return code of `mv`; on error, log and abort. |

---

## 6. Typical Execution / Debugging Steps  

1. **Manual start (development)**  
   ```bash
   export ENV_MODE=DEV   # or PROD as needed
   . /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_move_files_from_staging.properties
   bash -x move-mediation-scripts/bin/MNAAS_non_move_files_from_staging.sh
   ```
   *`-x`* prints each command; useful to verify variable expansion and file lists.

2. **Check lock status**  
   ```bash
   grep MNAAS_Script_Process_Id $MNAAS_move_files_from_staging_ProcessStatusFileName
   ps -p <pid>
   ```
   If a stale PID is present, remove the line or clear the file.

3. **Validate configuration**  
   ```bash
   env | grep MNAAS_   # ensure all required vars are set
   test -d "$MNAASMainStagingDirDaily" && echo "Staging OK"
   test -w "$MNAASMainStagingDirDaily_v2" && echo "Archive OK"
   ```

4. **Inspect logs**  
   ```bash
   tail -f $MNAAS_move_files_from_staging_LogPath
   ```

5. **Force a specific step** (e.g., only actives) by editing the status flag:  
   ```bash
   sed -i 's/MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=2/' $MNAAS_move_files_from_staging_ProcessStatusFileName
   ```

6. **Verify email/SDP** – check the local mail queue or the recipient inbox after a simulated missing‑file scenario.

---

## 7. External Config / Environment Dependencies  

| Variable (example) | Source | Use |
|--------------------|--------|-----|
| `MNAASMainStagingDirDaily` | properties file | Source staging directory. |
| `MNAASMainStagingDirDaily_v2` | properties file | Archive directory. |
| `MNAAS_move_files_from_staging_LogPath` | properties file | Path to the script’s log file. |
| `MNAAS_move_files_from_staging_ProcessStatusFileName` | properties file | Central status/flag file. |
| `ENV_MODE` | properties file / env | Controls whether SCP to Windows is performed (`PROD`). |
| `SSH_MOVE_*` (user, server, port, path) | properties file | Remote Windows server details for file copy. |
| `MOVE_TELENA_WINDOWS_TO_EMAIL`, `MOVE_TELENA_WINDOWS_CC_EMAIL` | properties file | Recipients for “window‑job not receiving” notifications. |
| `MNAAS_Telena_feed_windows_*` | properties file | Windows job metadata used in alert emails. |
| `MNAAS_support_team_email` | properties file | Support team CC for alerts. |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email`, `MOVE_DEV_TEAM` | properties file | SDP ticket creation metadata. |
| File‑name patterns (`Holland01_Traffic_Details_extn`, `Singapore01_Actives_extn`, etc.) | hard‑coded at top of script (e.g., `HOL_01_*TrafficDetails*.csv`) | Glob patterns used to locate files. |

---

## 8. Suggested Improvements (TODO)

1. **Centralise status‑file updates** – wrap all `sed -i` modifications in a single function that obtains an exclusive lock (`flock`) and writes a temporary file before atomically replacing the original. This eliminates race conditions and makes the file easier to audit.

2. **Add robust error handling for external commands** – after each `scp`, `mv`, and `mailx/sendmail` call, check `$?`. On failure, log the error, send an immediate alert, and abort the current step to avoid silent data loss. Consider retry logic for transient network failures.