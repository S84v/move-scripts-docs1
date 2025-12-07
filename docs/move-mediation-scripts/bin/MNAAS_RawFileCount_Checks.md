**File:** `move-mediation-scripts/bin/MNAAS_RawFileCount_Checks.sh`

---

## 1. Summary
This script is a daily validation step that checks the number of distinct raw traffic files loaded into the Hive/Impala table `mnaas.traffic_details_raw_daily` for the previous day. It updates a shared process‑status file, sends an informational email with the file count, and, on failure, creates an SDP ticket and logs the error. The script is designed to run under a cron schedule and coordinates its execution with other MNAAS daily validation jobs via a flag file.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **Raw_Filecount_validation** | - Sets the process‑status flag to “running”. <br> - Determines the target date (yesterday at 13:00). <br> - Refreshes Impala metadata for `traffic_details_raw_daily`. <br> - Executes a `SELECT COUNT(DISTINCT filename)` query for the target date. <br> - On success: emails the count, logs success, updates status file to *success*. <br> - On failure: triggers ticket/email routine and exits with error. |
| **terminateCron_successful_completion** | Resets the daily‑process flag to *idle*, records run time, logs completion, and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, optionally triggers ticket/email (only in PROD), records run time, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | Marks the job as *Failure* in the status file, checks whether an SDP ticket/email has already been created, and if not, records the ticket creation time and sets a flag to avoid duplicate tickets. |
| **email_and_SDP_ticket_triggering_step_validation** | Simple logger wrapper (currently only logs ticket creation). |
| **Main execution block** | - Ensures only a single instance runs (process‑count check). <br> - Reads the daily‑process flag from the status file. <br> - If flag is 0 or 1, runs the validation and terminates successfully. <br> - Otherwise logs a flag‑mismatch and terminates unsuccessfully. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_RawFileCount_Checks.properties` (defines all configurable paths, flags, script names, log locations, etc.) <br> • Environment variable `ENV_MODE` (determines whether ticket creation is performed). <br> • Impala cluster reachable at `192.168.124.90`. |
| **Outputs** | • Email (via `mailx`) sent to `MOVE_DEV_TEAM` (derived from `recepient_mail_Id`). <br> • Log entries appended to `${MNAAS_RawFileCount_Checks_LogPath}<date>`. <br> • Updated process‑status file (`${MNAAS_RawFileCount_Checks_ProcessStatusFileName}`) with flags, timestamps, job status, and ticket‑creation flag. |
| **Side‑Effects** | • May create an SDP ticket (external ticketing system – not shown in script, assumed to be invoked elsewhere). <br> • Alters shared status file used by other daily validation scripts. |
| **Assumptions** | • The properties file exists and defines all referenced variables (`MNAAS_RawFileCount_Checks_ProcessStatusFileName`, `MNAAS_RawFileCount_Checks_LogPath`, `MNAAS_Daily_Validations_Checks_ScriptName`, `MNAAS_RawFileCount_Checks_ScriptName`, `MOVE_DEV_TEAM`, etc.). <br>• The `impala-shell` binary is in the PATH and the user has permission to query the `mnaas` database. <br>• `mailx` is configured to send mail from `hdfs@mtthdoped01.intl.vsnl.co.in`. <br>• The script runs on a host with access to the shared filesystem where the status and log files reside. <br>• The date logic (`yesterday 13:00`) matches the ingestion window of the raw files. |

---

## 4. Integration Points & Interaction with Other Components

| Component | Interaction |
|-----------|-------------|
| **Process‑Status File** (`${MNAAS_RawFileCount_Checks_ProcessStatusFileName}`) | Shared with other daily validation scripts (e.g., `MNAAS_Daily_Validations_Checks.sh`). The flag `MNAAS_Daily_ProcessStatusFlag` indicates whether a validation job is running, allowing orchestrators to serialize jobs. |
| **Log Directory** (`${MNAAS_RawFileCount_Checks_LogPath}`) | Consumed by monitoring/alerting tools (e.g., Splunk, ELK) for operational visibility. |
| **Impala Cluster** (`192.168.124.90`) | Provides the source data (`traffic_details_raw_daily`). Any schema change or connectivity issue will affect this script. |
| **Mail System** (`mailx`) | Sends the daily file‑count report to the development and operations mailing lists (`move-mediation-it-ops@tatacommunications.com`, `hadoop-dev@tatacommunications.com`). |
| **SDP Ticketing System** | Triggered indirectly via `email_and_SDP_ticket_triggering_step`; the script only sets a flag; the actual ticket creation is assumed to be performed by a downstream process that watches the status file. |
| **Cron Scheduler** | The script is intended to be invoked by a cron entry (likely daily after raw file ingestion). The self‑check on process count prevents overlapping runs. |
| **Other Validation Scripts** | The flag file is read/written by sibling scripts (e.g., `MNAAS_PreValidation_Checks.sh`, `MNAAS_RAReports.sh`). Coordination ensures only one validation runs at a time. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Impala query failure (network, metadata, or permission)** | – Add retry logic around `impala-shell` calls. <br> – Monitor exit codes and alert on non‑zero status before proceeding to email. |
| **Duplicate email or SDP ticket** | – The status file flag `MNAAS_email_sdp_created` already prevents duplicates; ensure the flag file is on a reliable shared storage and is atomically updated (use `flock` if needed). |
| **Race condition with multiple script instances** | – Current `ps aux | grep -w $SCRIPT_NAME | wc -l` check can be fooled by fast restarts. Replace with a lock file (`flock -n /var/lock/MNAAS_RawFileCount.lck`) for robust mutual exclusion. |
| **Hard‑coded Impala host IP** | – Move the host address to the properties file to allow environment‑specific overrides. |
| **Date logic mismatch** (e.g., daylight‑saving changes) | – Use UTC timestamps or configurable ingestion window parameters instead of a fixed “yesterday 13:00”. |
| **Mail delivery failure** | – Capture `mailx` exit status; on failure, log and optionally retry or raise an alert. |
| **Process‑status file corruption** | – Validate the file format before reading/writing; consider using a simple JSON or key‑value store with atomic writes. |

---

## 6. Running & Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file exists and is readable.  
   - Verify `impala-shell`, `mailx`, and `logger` are in the PATH.  
   - Confirm the user has write permission to the log and status file locations.  

2. **Manual Execution**  
   ```bash
   export ENV_MODE=DEV   # or PROD
   ./MNAAS_RawFileCount_Checks.sh
   ```
   - Add `set -x` at the top of the script (uncomment) to trace command execution.  

3. **Debugging Steps**  
   - Check the generated log file: `tail -f ${MNAAS_RawFileCount_Checks_LogPath}$(date +_%F)`.  
   - Verify the status file values before and after run (`cat $MNAAS_RawFileCount_Checks_ProcessStatusFileName`).  
   - Run the Impala query manually to confirm it returns a count:  
     ```bash
     impala-shell -i 192.168.124.90 -B -q "SELECT COUNT(DISTINCT filename) FROM mnaas.traffic_details_raw_daily WHERE filename LIKE '%$(date -d "yesterday 13:00" +%Y%m%d)%';"
     ```  
   - Test email delivery: `echo "test" | mailx -s "test" -c $cc_List -r "$from_mail_Id" "$MOVE_DEV_TEAM"`.

4. **Cron Integration**  
   - Typical cron entry (run after raw file load, e.g., 02:00 AM):  
     ```cron
     0 2 * * * /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_RawFileCount_Checks.sh >> /dev/null 2>&1
     ```

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_RawFileCount_Checks_ProcessStatusFileName` | Path to the shared flag/status file. |
| `MNAAS_RawFileCount_Checks_LogPath` | Directory (and prefix) for daily log files. |
| `MNAAS_Daily_Validations_Checks_ScriptName` | Human‑readable script identifier used in log messages. |
| `MNAAS_RawFileCount_Checks_ScriptName` | Script name used for process‑count check. |
| `MOVE_DEV_TEAM` | Comma‑separated list of email recipients for the report. |
| `setparameter` | Additional key/value pairs (e.g., `ENV_MODE=PROD`) injected via `eval`. |
| `ENV_MODE` (environment) | Determines whether SDP ticket creation is performed (only in PROD). |

---

## 8. Suggested Improvements (TODO)

1. **Replace the ad‑hoc process‑count check with a robust file lock** (`flock`) to guarantee single‑instance execution across nodes.  
2. **Externalize the Impala host and database name** into the properties file and add validation of connectivity before running the query, reducing hard‑coded values and improving portability.  

---