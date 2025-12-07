**File:** `move-mediation-scripts/bin/MNAAS_Daily_Actives_tbl_Aggr.sh`  

---

## 1. High‑level Summary
This Bash script is the orchestrator for the **daily “actives” aggregation** step of the MNAAS (Mobile Network Access & Services) data‑move pipeline. It coordinates a Java aggregation job, refreshes the resulting Impala table, updates a shared process‑status file, and handles success/failure notification (email + SDP ticket). The script is intended to be run once per day by a scheduler (e.g., cron) and protects against concurrent executions.

---

## 2. Key Functions & Responsibilities  

| Function | Purpose | Important Actions |
|----------|---------|-------------------|
| **`MNAAS_insert_into_aggr_table`** | Launches the Java aggregation job and refreshes the Impala table. | * Updates process‑status flags in `$MNAAS_Daily_Actives_Aggr_ProcessStatusFileName`.<br>* Checks that the Java process (`$Dname_MNAAS_Insert_Daily_actives_tbl`) is not already running.<br>* Executes `java -cp … $Insert_Daily_Aggr_table …` with property & control files.<br>* On success runs `impala-shell -i $IMPALAD_HOST -q "$aggr_actives_daily_tblname_refresh"`.<br>* Logs outcome and calls termination helpers on failure. |
| **`terminateCron_successful_completion`** | Marks the run as successful and exits cleanly. | * Sets status flag = 0, job_status = Success, email flag = No, timestamps.<br>* Writes a final log entry and exits (0). |
| **`terminateCron_Unsuccessful_completion`** | Handles any error path. | * Logs failure, invokes `email_and_SDP_ticket_triggering_step`, exits (1). |
| **`email_and_SDP_ticket_triggering_step`** | Sends a failure notification and creates an SDP ticket (once per run). | * Sets job_status = Failure.<br>* Checks `MNAAS_email_sdp_created` flag; if “No”, sends two emails (plain mail + `mailx` SDP ticket) with log & status‑file references.<br>* Updates email‑sent flag and timestamps. |
| **Main script block** | Entry point – ensures single‑instance execution, decides whether to run aggregation. | * Reads previous PID from the status file; aborts if still running.<br>* Writes its own PID to the status file.<br>* Reads `MNAAS_Daily_ProcessStatusFlag`; if 0 or 1 → call `MNAAS_insert_into_aggr_table` then success termination; otherwise failure termination. |

*Note:* A commented‑out `MNAAS_insert_into_aggr_table_at_eod` function exists for a possible end‑of‑day variant but is not active.

---

## 3. Inputs, Outputs & Side Effects  

| Category | Item | Description |
|----------|------|-------------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Actives_tbl_Aggr.properties` | Provides all environment variables used throughout the script (paths, class names, hostnames, email lists, etc.). |
| **Process‑status file** | `$MNAAS_Daily_Actives_Aggr_ProcessStatusFileName` | Plain‑text key‑value file that stores flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_Script_Process_Id`, etc.). Updated by the script and read by other pipeline components. |
| **Control files** | `$MNAAS_Daily_Actives_AggrCntrlFileName` (and the commented‑out EOD variant) | Passed to the Java aggregation class; likely contains table/partition metadata. |
| **Java JAR** | `$MNAAS_Main_JarPath` + `$Insert_Daily_Aggr_table` | Executed to perform the heavy data aggregation. |
| **Impala** | `$IMPALAD_HOST`, `$aggr_actives_daily_tblname_refresh` | After Java job succeeds, an Impala `REFRESH` (or similar) statement is issued to make the new data visible. |
| **Log file** | `$MNAAS_DailyActivesAggrLogPath` | All `stderr` (via `exec 2>>`) and explicit `logger` calls are appended here. |
| **Email / SDP ticket** | `$MailId`, `$ccList`, `$SDP_ticket_from_email`, `$MOVE_DEV_TEAM` | Sent on failure (once per run). |
| **Side effects** | - Creation/refresh of an Impala table.<br>- Update of shared status file (used by downstream scripts).<br>- Generation of log entries.<br>- Possible email & ticket creation on error. |
| **Assumptions** | - All variables are correctly defined in the sourced properties file.<br>- The Java class `$Insert_Daily_Aggr_table` is idempotent and can be safely re‑run if needed.<br>- `impala-shell` is reachable from the host and the user has required privileges.<br>- Mail utilities (`mail`, `mailx`) are configured and can send external mail. |

---

## 4. Interaction with Other Components  

| Component | How this script connects |
|-----------|--------------------------|
| **Downstream scripts / reporting jobs** | They read the same process‑status file (`$MNAAS_Daily_Actives_Aggr_ProcessStatusFileName`) to determine whether the daily actives aggregation succeeded (`MNAAS_job_status=Success`) before proceeding. |
| **Upstream data loaders** | The control file (`$MNAAS_Daily_Actives_AggrCntrlFileName`) is populated by earlier ingestion steps (e.g., raw CDR load). This script expects it to be present and valid. |
| **Java aggregation class** | `Insert_Daily_Aggr_table` (inside `$MNAAS_Main_JarPath`) performs the actual transformation; this script only launches it and monitors exit code. |
| **Impala** | The script issues a refresh query (`$aggr_actives_daily_tblname_refresh`) to make the newly aggregated data visible to downstream analytics. |
| **Monitoring / alerting** | Failure triggers an email to `$MailId` and an SDP ticket via `mailx` to `insdp@tatacommunications.com`. The ticketing system likely consumes these messages for incident management. |
| **Scheduler (cron)** | The script is intended to be invoked by a cron entry; the PID‑check logic prevents overlapping runs. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale PID / orphaned process** – if the script crashes without clearing `MNAAS_Script_Process_Id`, subsequent runs will be blocked. | Daily aggregation may be skipped, causing data gaps. | Add a watchdog that validates the PID is still alive (e.g., `ps -p $PID`) before aborting; optionally implement a timeout/cleanup step. |
| **Concurrent Java jobs** – race condition if two scheduler instances start simultaneously before PID file is updated. | Duplicate loads, possible data corruption. | Ensure the PID check and write are atomic (e.g., use `flock` on the status file). |
| **Java job failure** – exit code non‑zero but the script may not capture all error details. | Incomplete aggregation, downstream jobs fail. | Capture Java stdout/stderr to a dedicated log and include it in the failure email. |
| **Impala refresh failure** – network or permission issue after successful Java job. | Data not visible to consumers despite successful aggregation. | Check the exit status of `impala-shell`; if non‑zero, treat as failure and trigger alert. |
| **Email/SDP spam** – repeated failures could flood inboxes. | Alert fatigue. | Implement a back‑off or deduplication (e.g., only send once per hour) and ensure `MNAAS_email_sdp_created` flag works correctly. |
| **Missing/incorrect properties** – undefined variables cause script to abort early. | Immediate failure, no logs. | Validate required variables at script start; exit with clear error if any are missing. |
| **Log file growth** – `exec 2>>` appends indefinitely. | Disk exhaustion. | Rotate logs via logrotate or implement size‑based truncation. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the properties file exists and all variables are defined.  
   - Verify Java, `impala-shell`, `mail`, `mailx` are in the PATH and functional.  
   - Confirm write permission on the log file and status file.  

2. **Manual execution** (for testing):  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files   # optional, just to be near files
   bash /path/to/MNAAS_Daily_Actives_tbl_Aggr.sh
   ```
   - The script runs with `set -x`, so each command is echoed to the log (and to stdout if run interactively).  

3. **Checking the run**  
   - Tail the log: `tail -f $MNAAS_DailyActivesAggrLogPath`.  
   - Verify the status file values (`grep MNAAS_job_status $MNAAS_Daily_Actives_Aggr_ProcessStatusFileName`).  
   - Confirm the Impala table is refreshed: `impala-shell -i $IMPALAD_HOST -q "SHOW TABLES LIKE '<table_name>';"`.  

4. **Debugging common issues**  
   - **PID conflict**: Look for stale PID in the status file; kill the process if necessary.  
   - **Java errors**: Check `$MNAAS_DailyActivesAggrLogPath` for Java stack traces; run the Java command manually with the same arguments to reproduce.  
   - **Impala errors**: Run the refresh query directly in `impala-shell` to see detailed error messages.  
   - **Email not sent**: Verify `mail`/`mailx` configuration, check `/var/log/maillog` for delivery attempts.  

5. **Cron integration**  
   - Typical crontab entry (run at 02:00 daily):  
     ```cron
     0 2 * * * /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Actives_tbl_Aggr.sh >> /dev/null 2>&1
     ```
   - Ensure the environment for cron loads the same profile that defines needed variables (or source the properties file as the script already does).  

---

## 7. External Config / Environment Variables  

| Variable (populated from properties) | Role |
|--------------------------------------|------|
| `MNAAS_DailyActivesAggrLogPath` | Path to the script’s error log (stderr redirected here). |
| `MNAAS_Daily_Actives_Aggr_ProcessStatusFileName` | Shared status/flag file used for coordination. |
| `Dname_MNAAS_Insert_Daily_actives_tbl` | Identifier used to detect a running Java aggregation process. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath` | Java classpath and JAR location. |
| `Insert_Daily_Aggr_table` | Fully‑qualified Java class name that performs aggregation. |
| `MNAAS_Property_filename` | Path to the same (or another) properties file passed to Java. |
| `MNAAS_Daily_Actives_AggrCntrlFileName` | Control file for the Java job (metadata). |
| `processname_actives` | Logical name passed to Java (used for logging inside Java). |
| `IMPALAD_HOST` | Hostname of the Impala daemon. |
| `aggr_actives_daily_tblname_refresh` | SQL statement (e.g., `REFRESH <db>.<table>`) executed after Java success. |
| `MNAASDailyActivesAggregationScriptName`, `MNAASDailyActivesAggrScriptName` | Human‑readable script identifiers used in log messages and email subjects. |
| `MailId`, `ccList` | Email recipients for failure notifications. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Parameters for the SDP ticket email (`mailx`). |
| `MNAAS_email_sdp_created` flag | Prevents duplicate ticket/email generation. |

If any of these are missing or malformed, the script will likely abort early or behave unpredictably.

---

## 8. Suggested Improvements (TODO)

1. **Atomic PID handling** – Wrap the PID read/write and status‑file updates in a `flock` block to guarantee single‑instance execution even under race conditions.  
2. **Centralised error handling** – Extract the repeated pattern of logging, status‑file update, and termination into a generic `fail_and_notify` function; this reduces duplication and makes future changes (e.g., adding Slack alerts) easier.  

---