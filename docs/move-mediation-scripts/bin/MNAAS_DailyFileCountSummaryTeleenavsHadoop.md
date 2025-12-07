**File:** `move-mediation-scripts/bin/MNAAS_DailyFileCountSummaryTeleenavsHadoop.sh`

---

## 1. Purpose (One‑paragraph summary)

This script generates a daily CSV report that compares the number of raw traffic files ingested by the Teleena system with the records stored in Hadoop (Impala table `mnaas.ra_file_count_rep`). It runs once per day (typically via cron), refreshes the Impala metadata, extracts the previous‑day statistics, emails the report to a predefined distribution list, and updates a shared *process‑status* file used by the broader MNAAS data‑move framework to indicate success or failure. In case of failure it triggers an SDP ticket and logs detailed diagnostic information.

---

## 2. Key Functions / Logical Units

| Function / Block | Responsibility |
|------------------|----------------|
| **Report_DailyFileCountSummaryTeleenavsHadoop** | - Sets status flag to *running* in the process‑status file.<br>- Determines *yesterday* (13:00 reference) date.<br>- Refreshes Impala table `mnaas.ra_file_count_rep`.<br>- Executes a SELECT query to retrieve file‑count metrics for that date and writes a CSV (`output_ra_report_YYYY-MM-DD.csv`).<br>- On success: chmod 777, email CSV via `mailx`, log success, mark job status *success*.<br>- On failure: invoke `email_and_SDP_ticket_triggering_step` and terminate with error. |
| **terminateCron_successful_completion** | Resets the daily process flag to *idle* (0), writes generic success metadata (run time, job status) to the status file, logs termination, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, optionally triggers SDP ticket (only in PROD), writes failure metadata, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Updates the status file to `MNAAS_job_status=Failure`. If an SDP ticket has not yet been created (`MNAAS_email_sdp_created=No`), records the run time, flips the flag to `Yes`, and logs ticket creation. |
| **email_and_SDP_ticket_triggering_step_validation** | Simple wrapper that only logs ticket creation (used for validation paths). |
| **Main guard block** (the `if [ \`ps aux| …\` ]` section) | Ensures only one instance runs, reads the process‑status flag, decides whether to invoke the report, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_CommonProperties.properties` – defines paths (`MNAASConfPath`, `MNAASLocalLogPath`), environment mode (`ENV_MODE`), script name variables (`MNAAS_Daily_Validations_Checks_ScriptName`), and other common parameters. |
| **Environment variables** | `setparameter` (evaluated after sourcing), `ENV_MODE` (used to decide whether to raise SDP tickets), any variables defined in the common properties file (e.g., `MNAASConfPath`). |
| **External services** | - **Impala** (host `192.168.124.90`) via `impala-shell`.<br>- **Mail server** via `mailx` (SMTP configured on the host).<br>- **SDP ticketing system** (invoked indirectly; script only logs creation, actual ticket generation is external). |
| **Files read** | - Process‑status file: `$MNAASConfPath/MNAAS_DailyFileCountSummaryTeleenavsHadoop_ProcessStatusFile`.<br>- Common properties file. |
| **Files written** | - CSV report: `$MNAASConfPath/Space_Monitoring/output_ra_report_YYYY-MM-DD.csv` (chmod 777).<br>- Log entries appended to `$MNAASLocalLogPath/MNAAS_DailyFileCountSummaryTeleenavsHadoop_Log_YYYY-MM-DD`.<br>- Updated process‑status file (multiple `sed` edits). |
| **Side‑effects** | - Sends email with CSV attachment.<br>- May create an SDP ticket (logged, external system).<br>- Updates shared status file used by other MNAAS scripts. |
| **Assumptions** | - Impala host reachable and `impala-shell` installed.<br>- `mailx` configured and able to send from `hdfs@mtthdoped01.intl.vsnl.co.in`.<br>- The status file exists and is writable by the script user.<br>- The `Space_Monitoring` directory exists under `$MNAASConfPath`.<br>- The script runs under a user with permission to modify the status file, write logs, and change file modes. |

---

## 4. Integration Points (How it connects to other components)

| Connection | Description |
|------------|-------------|
| **Process‑status file** (`MNAAS_DailyFileCountSummaryTeleenavsHadoop_ProcessStatusFile`) | Shared with other daily validation/orchestration scripts (e.g., `MNAAS_Daily_Validations_Checks_*`). The flag fields (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, etc.) are read/written by multiple scripts to coordinate execution order and error handling. |
| **Common properties** (`MNAAS_CommonProperties.properties`) | Provides global configuration (paths, environment mode, script identifiers) used across the entire MNAAS move‑framework. |
| **Impala table `mnaas.ra_file_count_rep`** | Populated by upstream ingestion pipelines (e.g., raw CDR loaders). The report validates that the Hadoop side reflects the Teleena raw file counts. |
| **Email distribution list** | Recipients (e.g., `lavanya.ippili@contractor.tatacommunications.com`, `RaghuRam.Peddi@contractor.tatacommunications.com`) are also used by other monitoring scripts, ensuring a unified notification channel. |
| **SDP ticketing** | While the script only logs ticket creation, other automation (perhaps a separate ticket‑creation daemon) monitors the log or status file to actually raise an SDP incident. |
| **Cron scheduler** | Typically invoked from a daily cron entry; the script’s self‑guard (`ps aux | grep -w $MNAAS_Daily_Validations_Checks_ScriptName`) prevents overlapping runs, a pattern used by many other MNAAS scripts. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Impala connectivity failure** (network, authentication) | No report generated; downstream monitoring blind. | - Add retry logic around `impala-shell`.<br>- Monitor exit code and raise SDP ticket immediately.<br>- Ensure host `192.168.124.90` is reachable from the execution node. |
| **CSV file permission/ownership issues** (chmod 777 may be too permissive) | Security exposure, possible overwrite failures. | - Use a more restrictive mode (e.g., 640) and ensure the consuming group has access.<br>- Verify directory ownership before execution. |
| **Mail delivery failure** (mailx mis‑configuration) | Stakeholders do not receive report; may miss data quality issues. | - Capture `mailx` exit status; on non‑zero, trigger SDP ticket.<br>- Periodically test mail delivery via a health‑check script. |
| **Concurrent executions** (status file race) | Corrupted status flags, duplicate emails, or missed runs. | - Use file locking (`flock`) around status‑file updates.<br>- Keep the existing `ps` guard but also verify lock acquisition. |
| **Missing/invalid configuration** (properties file absent or malformed) | Script aborts early, no logs. | - Add sanity checks after sourcing properties (e.g., verify required variables are non‑empty). |
| **Date calculation edge cases** (DST changes, leap seconds) | Wrong date used for query, leading to empty report. | - Use UTC consistently or validate the computed `yesterday_date` against a known calendar source. |
| **Hard‑coded Impala host** | Lack of flexibility across environments (DEV/PROD). | - Move host address to the common properties file. |

---

## 6. Typical Execution / Debugging Steps

1. **Manual run (dry‑run)**  
   ```bash
   export ENV_MODE=DEV   # or PROD as appropriate
   ./MNAAS_DailyFileCountSummaryTeleenavsHadoop.sh
   ```
   Observe console output; check that the log file `$MNAASLocalLogPath/..._Log_$(date +_%F)` contains “START TIME” and “END TIME”.

2. **Verify status file before execution**  
   ```bash
   cat $MNAASConfPath/MNAAS_DailyFileCountSummaryTeleenavsHadoop_ProcessStatusFile
   ```
   Ensure `MNAAS_Daily_ProcessStatusFlag=0` (idle).

3. **Check Impala query** (run manually)  
   ```bash
   impala-shell -i 192.168.124.90 -B -q "refresh mnaas.ra_file_count_rep"
   impala-shell -i 192.168.124.90 -B -q "select ... where file_date='2024-12-03';"
   ```
   Confirm rows are returned.

4. **Inspect generated CSV**  
   ```bash
   less $MNAASConfPath/Space_Monitoring/output_ra_report_$(date -d "yesterday" +_%F).csv
   ```

5. **Validate email delivery**  
   - Check the recipient inbox.<br>
   - Look at `/var/log/mail.log` (or equivalent) for `mailx` activity.

6. **Troubleshoot failures**  
   - Review the log file for the exact error line.<br>
   - If the script exits with status 1, confirm whether an SDP ticket was logged (`grep SDP $MNAASLocalLogPath/...`).<br>
   - Use `set -x` (uncomment the line near the top) to get a trace of each command.

7. **Cron integration**  
   - Ensure the cron entry runs under the same user that owns the status file and has the required environment (source the properties file if needed).  
   - Example cron line (run at 02:00 AM daily):
     ```cron
     0 2 * * * /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties && /path/to/MNAAS_DailyFileCountSummaryTeleenavsHadoop.sh >> /dev/null 2>&1
     ```

---

## 7. External Configurations & Environment Variables

| Variable / File | Origin | Usage |
|-----------------|--------|-------|
| `MNAAS_CommonProperties.properties` | Version‑controlled config repository | Provides `$MNAASConfPath`, `$MNAASLocalLogPath`, `$ENV_MODE`, `$MNAAS_Daily_Validations_Checks_ScriptName`, and other shared constants. |
| `setparameter` (evaluated via `eval $setparameter`) | Defined inside the common properties file (or a sourced script) | Supplies additional runtime parameters (e.g., JVM options, custom flags). |
| `MNAASConfPath` | Common properties | Base directory for status file and output CSV. |
| `MNAASLocalLogPath` | Common properties | Directory where daily log files are written. |
| `ENV_MODE` | Common properties / exported in the environment | Determines whether SDP ticket creation is performed (`PROD` only). |
| `MNAAS_Daily_Validations_Checks_ScriptName` | Common properties | Used for process‑guard (`ps aux | grep -w $MNAAS_Daily_Validations_Checks_ScriptName`). |
| `MNAAS_RawFileCount_Checks_ScriptName` | Set locally in this script | Not used elsewhere; may be a leftover. |
| `MNAAS_DailyFileCountSummaryTeleenavsHadoop_ProcessStatusFileName` | Constructed from `$MNAASConfPath` | Central status file for this job. |
| `output_file` | Constructed from `$MNAASConfPath/Space_Monitoring` | Destination CSV file. |
| Email addresses (`cc_List`, `from_mail_Id`, `recepient_mail_Id`) | Hard‑coded in script | Recipients of the daily report. |

---

## 8. Suggested Improvements (TODO)

1. **Introduce file locking** – Wrap all reads/writes to the process‑status file with `flock` (or a lock‑file) to eliminate race conditions when multiple MNAAS scripts are scheduled close together.

2. **Externalize host & query parameters** – Move the Impala host (`192.168.124.90`) and the SELECT statement into the common properties file or a dedicated query‑template file. This makes the script environment‑agnostic and easier to maintain across DEV/QA/PROD. 

---