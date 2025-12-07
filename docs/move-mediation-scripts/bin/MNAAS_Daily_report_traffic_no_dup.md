**File:** `move-mediation-scripts\bin\MNAAS_Daily_report_traffic_no_dup.sh`

---

## 1. High‑Level Summary
This script extracts the most recent three daily traffic partitions from the Hive/Impala table `traffic_details_raw_daily_with_no_dups`, writes each partition to a CSV file in a staging directory, and (when the environment flag permits) securely copies the CSVs to a downstream SFTP server. It maintains a lightweight process‑status file to coordinate concurrent runs, writes detailed logs, and on any failure raises an SDP ticket via `mailx`. The script is intended to be invoked by a scheduler (e.g., cron) as part of the daily “traffic‑no‑dup” data‑move pipeline.

---

## 2. Core Functions & Responsibilities

| Function / Block | Responsibility |
|------------------|----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_report_traffic_no_dup.properties`** | Sources all configurable variables (paths, DB credentials, hostnames, flags, etc.). |
| **`MNAAS_Daily_report_traffic_no_dup()`** | *Main worker*:<br>1. Marks process as “running” in the status file.<br>2. Captures the latest three unique partition dates.<br>3. For each partition:<br> a. Runs an `impala-shell` query to export the partition to `${partition}.csv`.<br> b. If `ENV_MODE="REMOVED"` (placeholder for production flag), copies the CSV to a remote SFTP server via `scp`.<br> c. Logs success/failure and removes the local CSV on success. |
| **`terminateCron_successful_completion()`** | Resets status‑file flags to “idle”, records success timestamp, writes final log entries, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion()`** | Logs failure, invokes ticket‑creation, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step()`** | Updates status file to “Failure”, checks if an SDP ticket has already been raised, and if not sends a formatted `mailx` message to the support mailbox, then marks the ticket‑sent flag. |
| **Main script block (PID guard)** | Prevents overlapping runs by checking the PID stored in the status file; if no other instance is active, writes its own PID, validates the process‑status flag (0 or 1), calls the main worker, and finally calls the success termination routine. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced from properties file)** | <ul><li>`MNAAS_Daily_report_traffic_no_dup_log_file_path` – absolute path to the log file.</li><li>`MNAAS_Daily_report_traffic_no_dup_ProcessStatusFileName` – text file storing flags, PID, timestamps, etc.</li><li>`STAGING_DIR` – local directory where CSVs are written.</li><li>`MNAAS_Traffic_Partitions_Daily_Uniq_FileName` – file containing all unique partition dates (one per line).</li><li>`MNAAS_Traffic_Partitions_Daily_Uniq_latest_3_FileName` – derived file holding the latest three dates.</li><li>`IMPALAD_HOST`, `dbname` – Impala connection details.</li><li>`ENV_MODE` – controls whether the `scp` step is executed (currently expects value `"REMOVED"` to trigger copy).</li><li>`SSH_MOVE_DAILY_SERVER`, `SSH_MOVE_DAILY_SERVER_PORT`, `SSH_MOVE_USER`, `SSH_MOVE_DAILY_PATH` – remote SFTP target.</li></ul> |
| **External services** | <ul><li>Impala daemon (`impala-shell`).</li><li>Remote SFTP server (via `scp`).</li><li>Mail system (`mailx`).</li></ul> |
| **Primary input data** | The Impala table `traffic_details_raw_daily_with_no_dups` filtered by `partition_date`. |
| **Generated output** | One CSV per processed partition, placed in `$STAGING_DIR` and optionally copied to the remote server. |
| **Side‑effects** | <ul><li>Updates the process‑status file (flags, PID, timestamps, job status, ticket‑sent flag).</li><li>Appends to the log file.</li><li>May create an SDP ticket (email) on failure.</li></ul> |
| **Assumptions** | <ul><li>All variables defined in the properties file are valid and accessible.</li><li>Impala query returns data quickly enough for the cron window.</li><li>SSH keys/passwordless `scp` are pre‑configured for the target host.</li><li>`mailx` is functional and the SMTP relay accepts the message.</li><li>The status file is writable by the script user and not corrupted.</li></ul> |

---

## 4. Interaction with Other Scripts / Components

| Connected Component | How the Interaction Occurs |
|---------------------|-----------------------------|
| **Other “MNAAS_*” daily scripts** (e.g., `MNAAS_Daily_Tolling_tbl_Aggr.sh`, `MNAAS_Daily_Validations_Checks.sh`) | Share the same *process‑status* directory and naming convention (`*_ProcessStatusFileName`). Coordination is implicit: each script writes its own flag values, but they may be monitored by a higher‑level orchestrator that checks overall job health. |
| **Properties repository** (`MNAAS_Property_Files/*.properties`) | Centralised configuration; any change to paths, hosts, or flags propagates to this script and its siblings. |
| **Impala/Hive data warehouse** | Reads from `traffic_details_raw_daily_with_no_dups`; downstream scripts may consume the CSVs placed on the remote SFTP server. |
| **Remote SFTP server** | Destination for the CSVs; downstream ingestion pipelines (e.g., reporting or analytics jobs) poll this location. |
| **SDP ticketing / email system** | Failure notifications are sent to `insdp@tatacommunications.com` and CC’d to internal groups; other monitoring tools may parse these emails. |
| **Cron scheduler** | The script is expected to be invoked by a daily cron entry; the PID guard prevents overlapping runs. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale or corrupted process‑status file** | Script may think another instance is running or mis‑interpret flags, causing missed runs. | Add sanity checks: verify PID existence, rotate status file daily, and alert if file is unreadable. |
| **Impala query failure / timeout** | No CSV generated → downstream pipelines stall. | Capture `impala-shell` exit code; on non‑zero exit, trigger failure path immediately. Consider adding a retry loop with exponential back‑off. |
| **`scp` failure (network, auth)** | CSV not delivered; data loss. | Verify SSH key validity before run, log detailed `scp` error, and optionally fallback to an alternative transfer method (e.g., `rsync` over VPN). |
| **`mailx` unavailable** | Failure not communicated to support, tickets not raised. | Test mail delivery in a pre‑run health‑check; if unavailable, write a local “alert” file and raise a system alarm (e.g., via `nagios`). |
| **Concurrent runs due to PID guard race condition** | Duplicate loads, duplicate files on remote server. | Use file locking (`flock`) around the PID write/read block, or move to a more robust scheduler (e.g., Airflow) that enforces singleton tasks. |
| **Hard‑coded `ENV_MODE="REMOVED"` check** | In production the copy step may never run if the flag is mis‑set. | Replace placeholder with a meaningful environment variable (e.g., `ENV_MODE=PROD`), and document accepted values. |
| **Missing partition file or empty latest‑3 list** | Loop reads nothing → script exits silently. | Validate that `$MNAAS_Traffic_Partitions_Daily_Uniq_latest_3_FileName` exists and contains at least one line; abort with error if not. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_report_traffic_no_dup.properties` is present and readable.  
   - Verify `impala-shell` is in `$PATH` and can connect to `$IMPALAD_HOST`.  
   - Confirm password‑less SSH to `$SSH_MOVE_DAILY_SERVER` works for the user `$SSH_MOVE_USER`.  
   - Test `mailx` by sending a dummy message.

2. **Manual Execution**  
   ```bash
   export ENV_MODE=REMOVED   # or appropriate value for your environment
   ./MNAAS_Daily_report_traffic_no_dup.sh
   ```
   - The script will log verbosely because of `set -x`.  
   - Check `$MNAAS_Daily_report_traffic_no_dup_log_file_path` for detailed output.  

3. **Debugging Tips**  
   - **PID Guard**: If you see “Previous … Process is running already”, inspect the status file and kill the stale PID if necessary.  
   - **Impala Query**: Run the generated query manually:  
     ```bash
     impala-shell -i $IMPALAD_HOST -d $dbname -q "SELECT * FROM traffic_details_raw_daily_with_no_dups where partition_date='2024-12-01'" -B -o /tmp/debug.csv
     ```  
   - **SCP**: Manually copy a test CSV to verify connectivity:  
     ```bash
     scp -P $SSH_MOVE_DAILY_SERVER_PORT test.csv $SSH_MOVE_USER@$SSH_MOVE_DAILY_SERVER:$SSH_MOVE_DAILY_PATH/
     ```  
   - **Email**: Force the failure path to test ticket creation:  
     ```bash
     terminateCron_Unsuccessful_completion
     ```

4. **Cron Integration**  
   - Typical crontab entry (run once daily at 02:30):  
     ```cron
     30 2 * * * /path/to/MNAAS_Daily_report_traffic_no_dup.sh >> /dev/null 2>&1
     ```
   - Ensure the user’s environment loads the properties file (the script does it internally).

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose | Typical Example |
|----------------------------|---------|-----------------|
| `MNAAS_Daily_report_traffic_no_dup_log_file_path` | Path for script log file. | `/var/log/mnaas/traffic_no_dup.log` |
| `MNAAS_Daily_report_traffic_no_dup_ProcessStatusFileName` | Shared status/flag file. | `/app/hadoop_users/MNAAS/status/traffic_no_dup.status` |
| `STAGING_DIR` | Local staging directory for CSVs. | `/data/mnaas/staging/traffic` |
| `MNAAS_Traffic_Partitions_Daily_Uniq_FileName` | Master list of all unique partition dates. | `/app/hadoop_users/MNAAS/partitions/traffic_uniq.txt` |
| `MNAAS_Traffic_Partitions_Daily_Uniq_latest_3_FileName` | File that will contain the latest three dates (generated by `head`). | `/app/hadoop_users/MNAAS/partitions/traffic_latest3.txt` |
| `IMPALAD_HOST` | Impala daemon host. | `impala01.tatacommunications.com` |
| `dbname` | Hive/Impala database name. | `mnaas_db` |
| `ENV_MODE` | Controls whether the `scp` block runs. | `REMOVED` (placeholder) |
| `SSH_MOVE_DAILY_SERVER`, `SSH_MOVE_DAILY_SERVER_PORT`, `SSH_MOVE_USER`, `SSH_MOVE_DAILY_PATH` | Remote SFTP target details. | `sftp.tata.com`, `22`, `move_user`, `/incoming/traffic` |
| `MNAAS_Daily_report_traffic_no_dup_scriptName` | Script name used in logs/emails (usually set in properties). | `MNAAS_Daily_report_traffic_no_dup.sh` |

If any of these are missing or malformed, the script will fail early (e.g., `sed` will error, `impala-shell` will not connect).

---

## 8. Suggested Improvements (TODO)

1. **Replace the placeholder `ENV_MODE="REMOVED"` check with a meaningful environment flag** (e.g., `ENV_MODE=PROD|DEV|TEST`) and document accepted values. This will prevent accidental omission of the `scp` step in production.

2. **Introduce robust error handling for the Impala query**: capture the exit status, retry on transient failures, and write a clear error message to the status file before invoking the ticket‑creation routine. This reduces false‑positive failures caused by temporary Hive/Impala hiccups.