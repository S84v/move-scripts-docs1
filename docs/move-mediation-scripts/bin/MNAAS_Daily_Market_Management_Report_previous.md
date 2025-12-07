**File:** `move-mediation-scripts\bin\MNAAS_Daily_Market_Management_Report_previous.sh`

---

## 1. High‑Level Summary
This Bash script is the daily driver for the *Market Management Report* (MMR) pipeline. It coordinates a Spark job that extracts and transforms the previous month’s market‑management data, refreshes an Impala summary view, and records the run status in a shared process‑status file. The script is intended to be invoked by a scheduler (e.g., cron) and includes built‑in guarding against concurrent executions, status flag handling, HDFS permission fixes, and automated failure notification via email/SLA ticket creation.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **load_data_into_mmr_table** | *Pre‑run*: set process‑status flag to “running”. <br>*Compute*: derive `start_date`, `end_date`, `year_month` for the previous month.<br>*Prepare*: chmod HDFS directories used by the Spark job.<br>*Execute*: launch the Spark job (`$MNAAS_Daily_Market_Mgmt_Report_Pyfile`) with the computed dates.<br>*Post‑run*: on success run an Impala refresh query, log success, and invoke `terminateCron_successful_completion`; on failure log and invoke `terminateCron_Unsuccessful_completion`. |
| **terminateCron_successful_completion** | Reset status flags to “idle/Success”, write final log entries, and exit with status 0. |
| **terminateCron_Unsuccessful_completion** | Log failure, trigger `email_on_reject_triggering_step`, and exit with status 1. |
| **email_on_reject_triggering_step** | Build a minimal RFC‑822 style body and send an incident email (via `mailx`) to the SDP ticket mailbox, then log that the ticket/email was generated. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Daily_Market_Management_Report.properties` – defines all environment variables used throughout the script (paths, hostnames, queries, email addresses, etc.). |
| **Process‑Status File** (`$MNAAS_Daily_Market_Management_Report_ProcessStatusFileName`) | Read/write: stores PID, process name, flag (`MNAAS_Daily_ProcessStatusFlag`), job status, email‑trigger flag. Acts as a lightweight lock & status monitor for downstream scripts. |
| **Log File** (`$MNAAS_Daily_Market_Mgmt_Report_logpath`) | Append‑only log of start/end timestamps, success/failure messages, and any error output from Spark/Impala. |
| **HDFS Directories** | `/user/hive/warehouse/mnaas.db/market_management_report_sims/*` – permissions are set to `777` before Spark runs. |
| **Spark Job** | Executed via `spark-submit` on YARN; consumes the Python file `$MNAAS_Daily_Market_Mgmt_Report_Pyfile` and three date arguments. |
| **Impala Query** | `$MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh` – run with `impala-shell` to refresh a materialised view or table after Spark succeeds. |
| **Email/SLA Ticket** | Sent via `mailx` to `insdp@tatacommunications.com` (and CCs `$T0_email`) when the job fails. |
| **Assumptions** | • All variables defined in the properties file are present and valid.<br>• YARN, Spark, Impala, and HDFS services are reachable from the host.<br>• `mailx` is configured for outbound SMTP.<br>• The process‑status file is on a shared, writable filesystem accessible to all related scripts. |

---

## 4. Interaction with Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **`MNAAS_Daily_Market_Management_Report.sh`** (current version) | Likely a newer/maintained wrapper that calls the same Python job; this “_previous_” script may be kept for rollback or historical reference. |
| **KYC Feed Scripts** (`MNAAS_Daily_KYC_Feed_Loading.sh`, `MNAAS_Daily_KYC_Feed_tbl_loading.sh`) | Share the same process‑status file convention and logging infrastructure; they may be scheduled sequentially or in parallel depending on business rules. |
| **Scheduler (cron / Oozie / Airflow)** | Invokes this script daily; the script itself checks the PID stored in the status file to avoid overlapping runs. |
| **HDFS / Hive Metastore** | The Spark job reads/writes tables under `mnaas.db.market_management_report_sims`. |
| **Impala Daemon** (`$IMPALAD_HOST`) | Receives the refresh query to make the new data visible to downstream reporting tools. |
| **SDP Ticketing System** | Failure email is routed to `insdp@tatacommunications.com`, which likely creates a ticket automatically. |
| **Monitoring / Alerting** | Log entries and the status file are consumable by external monitoring tools (e.g., Splunk, Nagios) to raise alerts on non‑zero exit codes. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – PID check may be stale if the previous process crashed without clearing the file. | Implement a lock file with a timeout or use `flock`. Verify the PID is still alive (`kill -0 $PID`). |
| **Process‑status file corruption** – Multiple scripts edit the same file via `sed -i`. | Switch to atomic updates (write to temp file then `mv`). Consider using a lightweight DB (SQLite) or a key‑value store (Redis) for status. |
| **Hard‑coded `chmod 777`** – Overly permissive, may expose data. | Restrict to required user/group (`chmod 750`) and ensure the Spark job runs under the correct user. |
| **Spark job failure not captured** – Only `$?` after `spark-submit` is checked; Spark may exit with non‑zero but still produce partial data. | Capture Spark logs, parse for `FAILED` stages, and add a sanity check on output row counts before proceeding to Impala refresh. |
| **Impala query failure** – Not checked; script proceeds to success path. | Capture exit code of `impala-shell` and treat non‑zero as failure, invoking the error path. |
| **Email delivery failure** – `mailx` may silently drop messages. | Verify SMTP return code, add retry logic, and optionally log to a ticketing API directly. |
| **Missing/invalid properties** – Script will abort with obscure errors. | Add a validation block after sourcing the properties file that checks for required variables and exits early with a clear message. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Market_Management_Report.properties` is present and contains all required variables.  
   - Verify that the user executing the script has write permission to the status file and log path.  
   - Confirm Spark, Impala, HDFS, and `mailx` are reachable.

2. **Manual Execution**  
   ```bash
   # As the appropriate user
   cd /path/to/move-mediation-scripts/bin
   ./MNAAS_Daily_Market_Management_Report_previous.sh
   ```
   The script already runs with `set -x`, so each command will be echoed to stdout and logged.

3. **Debugging Tips**  
   - **Check the status file** before and after run: `cat $MNAAS_Daily_Market_Management_Report_ProcessStatusFileName`.  
   - **Inspect the log**: `tail -f $MNAAS_Daily_Market_Mgmt_Report_logpath`.  
   - **Force a failure** (e.g., rename the Python file) to verify the email path.  
   - **Validate dates**: add `echo "Start=$start_date End=$end_date YM=$year_month"` inside `load_data_into_mmr_table` if needed.  
   - **Spark logs**: locate the YARN application ID printed by `spark-submit` and view its logs via `yarn logs -applicationId <appId>`.

4. **Exit Codes**  
   - `0` – Successful completion (status file set to `Success`).  
   - `1` – Failure (email sent, status file set to `Failure`).  

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_Daily_Market_Management_Report_ProcessStatusFileName` | Path to the shared status/lock file. |
| `MNAAS_Daily_Market_Mgmt_Report_logpath` | Log file location. |
| `MNAAS_Daily_Market_Mgmt_Report_Pyfile` | Full path to the Spark Python driver script. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon for the refresh query. |
| `MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh` | SQL statement executed after Spark succeeds. |
| `MNAAS_Daily_Market_Mgmt_Report_Script` | Human‑readable script identifier used in logs/emails. |
| `SDP_ticket_from_email` | “From” address for the failure email. |
| `T0_email` | CC address for the failure email. |
| `MNAAS_FlagValue` (derived) | Controls whether the job may start (0 or 1). |
| *Other variables* (e.g., `MNAAS_FlagValue`, `MNAAS_Monthly_Traffic_Nodups_Script`) | Used for logging or conditional checks; defined in the same properties file. |

---

## 8. Suggested Improvements (TODO)

1. **Atomic Status Updates & Locking**  
   Replace the series of `sed -i` edits with a single atomic write (e.g., `printf` to a temp file then `mv`) and protect the whole critical section with `flock` to avoid race conditions.

2. **Enhanced Error Handling for Impala Refresh**  
   Capture the exit status of `impala-shell`; if non‑zero, treat the overall run as failed and trigger the email path. Also log the Impala output for post‑mortem analysis.

*(Additional enhancements such as externalising the email template, parameterising HDFS permission mode, and adding a health‑check endpoint could be considered in later iterations.)*