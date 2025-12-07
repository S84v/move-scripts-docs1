**File:** `move-mediation-scripts/bin/MNAAS_LateLandingCDRs_Check.sh`  

---

## 1. Purpose (one‑paragraph summary)

The script is a daily validation job that inspects the `mnaas.traffic_details_raw_daily` Impala table for “late‑landing” Call Detail Records (CDRs) – records whose file‑name timestamp indicates they arrived after the expected processing window (hour > 03) and belong to a previous month. If any such CDRs are found for the previous day, the script generates a CSV report, emails it to a predefined distribution list, and updates a shared process‑status file. It also integrates with the broader “MNAAS Daily Validations” framework, handling start‑up flag checks, logging, and failure escalation via SDP ticket creation.

---

## 2. Key Functions / Logical Blocks

| Function / Block | Responsibility |
|------------------|----------------|
| **`Late_Landing_CDRs_Check()`** | * Queries Impala for late‑landing CDRs (yesterday’s date, filename hour > 03, partition date older than last day of previous month). <br>* Writes result set to a CSV file (`LateLanding_CDRs_YYYY‑MM‑DD.csv`). <br>* Sends email with attachment when rows exist, updates process‑status file to *success*. |
| **`terminateCron_successful_completion()`** | * Resets the daily process flag to *0* (idle), writes final status (`Success`) and runtime to the status file, logs completion, and exits 0. |
| **`terminateCron_Unsuccessful_completion()`** | * Logs failure, optionally triggers SDP ticket & alert email when `ENV_MODE=PROD`, logs end‑time and exits 1. |
| **`email_and_SDP_ticket_triggering_step()`** | * Marks the status file as `Failure`. <br>* Checks if an SDP ticket/email has already been generated (`MNAAS_email_sdp_created`). If not, updates the flag, logs ticket creation. |
| **`email_and_SDP_ticket_triggering_step_validation()`** | * Simple wrapper that only logs ticket creation (used only for validation paths). |
| **Main guard (`if [ \`ps aux| grep -w $MNAAS_LateLandingCDRs_Check_ScriptName | wc -l\` -le 3 ]`)** | * Prevents concurrent runs by checking the process count. <br>* Reads the shared flag (`MNAAS_Daily_ProcessStatusFlag`) to decide whether to start the check or abort. <br>* Drives the overall flow: start → `Late_Landing_CDRs_Check` → success/unsuccessful termination. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `MNAAS_LateLandingCDRs_Check.properties`. Expected variables (non‑exhaustive): <br>• `MNAAS_LateLandingCDRs_Check_ProcessStatusFileName` – path to a key‑value status file. <br>• `MNAAS_LateLandingCDRs_Check_LogPath` – directory for daily log files. <br>• `MNAAS_Daily_Validations_Checks_ScriptName`, `MNAAS_LateLandingCDRs_Check_ScriptName` – script identifiers used in logs and flag checks. <br>• `MOVE_DEV_TEAM` – comma‑separated email list for the primary recipients. <br>• `ENV_MODE` – `PROD` or other (controls SDP ticket creation). |
| **External Services** | • **Impala** (`impala-shell -i 192.168.124.93`) – runs two queries (metadata refresh + data select). <br>• **Mail** (`mailx`) – sends CSV attachment. <br>• **SDP ticketing** – invoked via a custom internal command (not shown) when failures occur in PROD. |
| **File System** | • Reads/writes the **status file** (plain text key‑value). <br>• Writes CSV report to `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/LateLanding_CDRs/LateLanding_CDRs_YYYY‑MM‑DD.csv`. <br>• Writes daily log to `$MNAAS_LateLandingCDRs_Check_LogPathYYYY‑MM‑DD`. |
| **Outputs** | • CSV report (if any rows). <br>• Email with subject “Alert : Late Landing CDRs - Traffic Details Table.” <br>• Updated status file (flags: `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_job_ran_time`, etc.). |
| **Assumptions** | • Impala host `192.168.124.93` is reachable and the `mnaas` database/schema exists. <br>• The `traffic_details_raw_daily` table contains a `filename` column with a timestamp pattern at positions 35‑42. <br>• The process‑status file is the single source of truth for concurrency control across the daily validation suite. <br>• The `mailx` command is configured with a working SMTP relay. |

---

## 4. Interaction with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **MNAAS_Daily_Validations_Checks.sh** (parent orchestrator) | This script is invoked as one of several validation steps. The orchestrator sets the shared status file and expects the flag `MNAAS_Daily_ProcessStatusFlag` to be toggled by each child script. |
| **Other validation scripts** (e.g., `MNAAS_GBS_Journal_Sqoop_Load.sh`, `MNAAS_HDFSSpace_Checks.sh`) | All share the same status file pattern (`MNAAS_*_ProcessStatusFileName`). They rely on the same flag‑checking guard to avoid overlapping runs. |
| **Impala metastore** | The script issues a `refresh` on the `traffic_details_raw_daily` table to ensure the latest partitions are visible before querying. |
| **Mail distribution list** | Uses the `MOVE_DEV_TEAM` variable (defined in the properties file) to address the primary recipients; additional CC list is hard‑coded in the script. |
| **SDP ticketing system** | Failure path (`email_and_SDP_ticket_triggering_step`) writes a flag in the status file and logs ticket creation; the actual ticket creation command is abstracted away (likely a wrapper script called elsewhere). |
| **Cron scheduler** | The script is intended to be run via a daily cron job; the guard `ps aux | grep -w $MNAAS_LateLandingCDRs_Check_ScriptName` prevents overlapping executions. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Impala connectivity / query failure** | No report generated; downstream processes may assume success. | – Add retry logic around `impala-shell` calls. <br>– Capture non‑zero exit codes and log detailed error messages. |
| **Large result set causing memory/timeout** | Script may hang or crash, leaving stale flag state. | – Impose a row‑limit or paginate the query. <br>– Monitor query execution time; alert if > threshold. |
| **Email delivery failure** | Stakeholders not notified of late CDRs. | – Verify `mailx` return code; fallback to an alternative notification channel (e.g., Slack, ticket). |
| **Concurrent execution due to race condition** | Two instances could both read flag=0 and both run, corrupting status file. | – Use file‑based locking (`flock`) in addition to the process‑count guard. |
| **Incorrect status‑file path or permission** | Script cannot update flags → subsequent jobs may be blocked. | – Ensure the status file is owned by the script user and has write permissions. <br>– Add sanity check at start (`test -w $file`). |
| **Hard‑coded email addresses** | Maintenance overhead; missing new recipients. | – Move all recipient lists to the properties file. |
| **Time‑zone / date logic** | Using `date -d "yesterday 13:00"` may produce unexpected dates around DST changes. | – Document the intended timezone; consider using UTC consistently. |

---

## 6. Typical Execution / Debugging Steps

1. **Preparation**  
   - Verify that the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_LateLandingCDRs_Check.properties` is present and contains all required variables.  
   - Ensure the script user has execute permission (`chmod +x MNAAS_LateLandingCDRs_Check.sh`).  

2. **Manual Run (for testing)**  
   ```bash
   export ENV_MODE=DEV   # optional, forces non‑PROD path
   ./MNAAS_LateLandingCDRs_Check.sh
   ```
   - Observe console output; check the generated log file under `$MNAAS_LateLandingCDRs_Check_LogPath$(date +_%F)`.  
   - If a CSV is produced, verify its content and that the email was received.  

3. **Debugging**  
   - Enable shell tracing by uncommenting `set -x` near the top of the script.  
   - Check the exit status of each `impala-shell` command: `echo $?` after the call.  
   - If the script aborts in the “unsuccessful” path, inspect the status file for `MNAAS_job_status=Failure` and the log for the SDP ticket creation message.  

4. **Cron Integration**  
   - The cron entry typically looks like:  
     ```cron
     30 4 * * * /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_LateLandingCDRs_Check.sh >> /var/log/mnaas/late_landing_cron.log 2>&1
     ```  
   - Verify that the cron user’s environment loads the required PATH for `impala-shell` and `mailx`.  

5. **Post‑run Validation**  
   - Confirm that the status file flags are reset (`MNAAS_Daily_ProcessStatusFlag=0`, `MNAAS_job_status=Success`).  
   - Ensure the daily log file contains “START TIME” and “END TIME” entries.  

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Description | Typical Value |
|----------------------------|-------------|---------------|
| `MNAAS_LateLandingCDRs_Check_ProcessStatusFileName` | Path to the shared key‑value status file used for concurrency control and status reporting. | `/app/hadoop_users/MNAAS/status/late_landing_status.cfg` |
| `MNAAS_LateLandingCDRs_Check_LogPath` | Directory (plus prefix) for daily log files. | `/app/hadoop_users/MNAAS/logs/late_landing/late_landing_log` |
| `MNAAS_Daily_Validations_Checks_ScriptName` | Human‑readable name used in log messages. | `MNAAS_Daily_Validations_Checks.sh` |
| `MNAAS_LateLandingCDRs_Check_ScriptName` | Name of this script (used for process‑count guard). | `MNAAS_LateLandingCDRs_Check.sh` |
| `MOVE_DEV_TEAM` | Comma‑separated list of primary email recipients. | `team@example.com` |
| `ENV_MODE` | Execution environment; controls SDP ticket creation. | `PROD` or `DEV` |
| `from_mail_Id` | Sender address for notification emails. | `hdfs@mtthdoped01.intl.vsnl.co.in` |
| `cc_List` | CC recipients (hard‑coded in script). | `anusha.gurumurthy@contractor.tatacommunications.com,...` |

*If any of these variables are missing or empty, the script will likely fail or behave unpredictably.*

---

## 8. Suggested Improvements (TODO)

1. **Add Robust Error Handling for Impala Queries**  
   - Capture the exit status of each `impala-shell` call, log the error, and trigger the failure path (`terminateCron_Unsuccessful_completion`) instead of proceeding with empty/partial data.

2. **Externalize All Email Addresses to the Properties File**  
   - Move `cc_List` and `from_mail_Id` out of the script body to avoid hard‑coding and simplify maintenance across environments.

*(Additional enhancements such as implementing `flock` for locking, parameterising the Impala host, and using a configuration management tool (e.g., Ansible) for property deployment are also worth considering.)*