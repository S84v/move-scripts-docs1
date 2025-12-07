**File:** `move-mediation-scripts/bin/MNAAS_TMC_customer_daily_report.sh`

---

## 1. Purpose (One‑paragraph summary)

This script generates the daily “TMC Customer” report for the Move‑Mediation platform. It reads a Hive query (defined in a properties file), executes it via Beeline, writes the result set to an HDFS staging directory, concatenates the files into a single CSV, adds a header line, and copies the final file to a production location (and an “EV” backup). Throughout the run it updates a shared process‑status file, logs progress to an aggregation log, and on failure sends a formatted email to the SDP ticketing mailbox. The script is intended to be invoked by a scheduler (e.g., cron) and includes safeguards to prevent concurrent executions.

---

## 2. Key Functions / Components

| Function / Symbol | Responsibility |
|-------------------|----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_TMC_customer_daily_report.properties`** | Loads all configurable variables (HDFS paths, Hive connection, query string, log files, email recipients, etc.). |
| **`MNAAS_TMC_customer_daily_report_table()`** | Core workflow: sets “running” flag, ensures single instance, creates HDFS directory, runs Hive query via Beeline, assembles output CSV, copies to EV location, logs success/failure. |
| **`terminateCron_successful_completion()`** | Resets status flags, cleans up HDFS staging files, writes final success entries to the aggregation log, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion()`** | Logs failure, triggers email notification, and exits with status 1. |
| **`email_on_reject_triggering_step()`** | Builds a minimal RFC‑822 body and sends a mailx message to `insdp@tatacommunications.com` (SDP ticketing) with predefined categories and summary. |
| **Main program block** | Reads previous PID from the status file, checks if a prior instance is still alive, writes the current PID, validates the daily‑process flag (0 or 1), invokes the table function, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs, Side‑effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • All variables defined in `MNAAS_TMC_customer_daily_report.properties` (e.g., `MNAAS_Active_feed_hdfs_location`, `MNAAS_TMC_customer_daily_report_ProcessStatusFileName`, `HIVE_IP`, `Query`). <br>• Hive table data accessed via the supplied `Query`. |
| **Outputs** | • CSV file at `$MNAAS_Active_feed_location` (daily report). <br>• Copy of the CSV at `$MNAAS_Active_feed_location_EV`. <br>• Log entries appended to `$MNAAS_TMC_customer_daily_report_AggrLogPath`. |
| **Side‑effects** | • Creates/updates HDFS directory `$MNAAS_Active_feed_hdfs_location`. <br>• Changes HDFS ownership to `hdfs:supergroup`. <br>• Removes all files under the HDFS staging directory on successful completion. <br>• Sends an email via `mailx` on failure. |
| **External Services / Systems** | • **Hive/Beeline** (JDBC URL built from `$HIVE_IP`). <br>• **HDFS** (commands `hdfs dfs`). <br>• **Local filesystem** (writes CSV, chmod 777). <br>• **Mail server** (used by `mailx`). |
| **Assumptions** | • The properties file exists and contains all required variables. <br>• The Hive server is reachable and the query is syntactically correct. <br>• The user executing the script has HDFS write permissions and can `chown` to `hdfs:supergroup`. <br>• `mailx` is configured and the destination mailbox is valid. <br>• The status file is a simple key‑value text file used by multiple Move‑Mediation jobs. |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Process‑status file** (`$MNAAS_TMC_customer_daily_report_ProcessStatusFileName`) | Shared with other Move‑Mediation jobs (e.g., `MNAAS_SimInventory_tbl_Load.sh`). It stores flags (`MNAAS_Daily_ProcessStatusFlag`), the script PID, and the last run name. This enables a global “single‑run‑per‑day” coordination. |
| **HDFS staging directory** (`$MNAAS_Active_feed_hdfs_location`) | Other downstream jobs may consume the generated CSV from `$MNAAS_Active_feed_location` (e.g., reporting pipelines, data‑warehouse loaders). |
| **Aggregation log** (`$MNAAS_TMC_customer_daily_report_AggrLogPath`) | Consolidated log used by monitoring/alerting tools; other scripts may tail or parse this file for health checks. |
| **Email notification** | Failure emails are routed to the SDP ticketing system (`insdp@tatacommunications.com`). The same mailbox is used by many Move‑Mediation scripts for incident tracking. |
| **Cron scheduler** | The script is expected to be invoked by a cron entry (similar to other `MNAAS_*.sh` scripts). The PID‑check logic prevents overlapping runs when the cron frequency is higher than the job duration. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – PID check uses `ps aux | grep -w $MNAAS_TMC_customer_daily_report_table` which may match the grep process itself, leading to false positives. | Replace with `pgrep -f "$MNAAS_TMC_customer_daily_report_table"` or store the PID in a lock file and use `flock`. |
| **Stale PID in status file** – If the script crashes, the PID remains, blocking future runs. | Add a sanity check on the PID’s start time (`ps -p $PID -o lstart=`) and clear it if the process is older than a threshold. |
| **Hard‑coded `chmod 777`** – Overly permissive file permissions. | Use a more restrictive mode (e.g., `0640`) and rely on group ownership for access. |
| **Unvalidated properties** – Missing or malformed variables cause silent failures. | Add a validation block after sourcing the properties file that aborts with a clear error if any required variable is empty. |
| **Email failure** – `mailx` may not deliver, leaving operators unaware of failures. | Capture the exit status of `mailx`; if non‑zero, write an additional entry to the aggregation log and optionally trigger a secondary alert (e.g., syslog). |
| **Hive query failure** – No retry logic; a transient Hive outage aborts the whole job. | Implement a simple retry loop (e.g., up to 3 attempts with exponential back‑off) before marking the run as failed. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `MNAAS_TMC_customer_daily_report.properties` is present and populated.  
   - Verify Hive connectivity (`beeline -u "jdbc:hive2://$HIVE_IP/default"`).  
   - Confirm HDFS write permission for the staging directory.  
   - Test `mailx` by sending a test message.

2. **Manual Execution**  
   ```bash
   export setparameter="set -e"   # or whatever the calling environment provides
   ./MNAAS_TMC_customer_daily_report.sh
   ```
   - The script prints the previous PID and the running‑status count (useful for quick sanity).  
   - All log output is appended to `$MNAAS_TMC_customer_daily_report_AggrLogPath`.

3. **Debug Mode**  
   - The script already enables `set -x` (trace) at the top; you can increase verbosity by adding `set -v` or by exporting `BASH_XTRACEFD=5` to redirect traces to a separate file.  
   - To isolate the Hive step, run the Beeline command directly:
     ```bash
     beeline -u "jdbc:hive2://$HIVE_IP/default" -e "$Query"
     ```

4. **Checking Results**  
   - Verify the CSV exists: `ls -l $MNAAS_Active_feed_location`.  
   - Confirm the header line: `head -1 $MNAAS_Active_feed_location`.  
   - Ensure the HDFS staging area is empty after success: `hdfs dfs -ls $MNAAS_Active_feed_hdfs_location`.

5. **Failure Investigation**  
   - Review the aggregation log for the “failed” entry and the timestamp.  
   - Check the email inbox of `insdp@tatacommunications.com` for the incident ticket.  
   - Look at Hive server logs if the query failed.  
   - Verify that the status file still contains `MNAAS_Daily_ProcessStatusFlag=1`; if so, manually reset it to `0` before re‑running.

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Role |
|----------------------------|------|
| `MNAAS_TMC_customer_daily_report_ProcessStatusFileName` | Path to the shared status file (PID, flags, process name). |
| `MNAAS_Active_feed_hdfs_location` | HDFS directory used as staging area for the report files. |
| `MNAAS_Active_feed_location` | Local filesystem path where the final CSV is written. |
| `MNAAS_Active_feed_location_EV` | Backup copy location (EV = “Enterprise View” or similar). |
| `MNAAS_TMC_customer_daily_report_AggrLogPath` | Aggregated log file for this job. |
| `MNAAS_TMC_customer_daily_report_Script` | Human‑readable script identifier used in logs/emails. |
| `HIVE_IP` | Hostname/IP of the HiveServer2 instance. |
| `Query` | The HiveQL statement that extracts the daily TMC customer data. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email metadata used by `email_on_reject_triggering_step`. |
| `setparameter` (environment) | Passed from the calling framework; typically contains generic shell options. |

If any of these variables are missing or empty, the script will likely abort or behave unpredictably.

---

## 8. Suggested Improvements (TODO)

1. **Robust locking mechanism** – Replace the current PID‑and‑`ps` based guard with `flock` on a lock file to avoid race conditions and stale locks.
2. **Parameter validation block** – Add a function that checks each required variable after sourcing the properties file and exits early with a clear error message if any are undefined.

---