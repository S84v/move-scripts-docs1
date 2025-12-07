**File:** `move-mediation-scripts/bin/MNAAS_sqoop_bareport.sh`  

---

## 1. High‑Level Summary
This script extracts the **CBF GNV invoice generation** view (`cbf_auto_move_details_mview`) from a PostgreSQL database, loads it into a Hadoop HDFS staging directory, and then populates a Hive table (and refreshes the corresponding Impala table). It is guarded by a simple PID‑lock to avoid concurrent runs, updates a shared *process‑status* file, writes detailed logs, and on failure sends an email and creates an SDP ticket. The script is typically invoked by a daily/periodic cron job as part of the “move” data‑pipeline that moves billing‑related data from source systems into the analytics data‑lake.

---

## 2. Key Functions & Responsibilities

| Function / Block | Responsibility |
|------------------|----------------|
| **`CBF_GNV_Invoice_generation_table_Sqoop`** | *Core ETL* – clears previous HDFS files, truncates the Hive target table, runs a Sqoop import (single mapper, `^` delimiter, `append` mode), loads the HDFS files into Hive, refreshes Impala, and logs each step. |
| **`terminateCron_successful_completion`** | Marks the process as **Success** in the status file, resets the daily flag, logs successful termination, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, updates the status file with the run timestamp, invokes the email/SDP routine, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Sends a failure notification email (using `mail`) and flags that an SDP ticket has been created to avoid duplicate alerts. |
| **Main program (PID lock & flag check)** | Prevents overlapping executions by checking a stored PID, writes the current PID, validates the daily flag (`0` or `1`), and dispatches the core function. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties` – defines all variables used throughout the script (DB connection, HDFS paths, Hive table names, log file name, status‑file path, mail recipients, Impala host, etc.). |
| **Environment variables** | None required beyond those defined in the properties file; the script assumes a Bash environment with `hadoop`, `hive`, `sqoop`, `impala-shell`, `logger`, and `mail` binaries on the PATH. |
| **External services** | • PostgreSQL (source DB) <br>• Hadoop HDFS (staging directory) <br>• Hive Metastore (target table) <br>• Impala (metadata refresh) <br>• Local syslog (`logger`) <br>• SMTP server (via `mail`) |
| **Primary output** | Populated Hive table `${dbname}.${CBF_GNV_Invoice_generation_table_name}` (and refreshed Impala table). |
| **Log output** | Append‑only log file `$CBF_GNV_Invoice_generation_table_SqoopLogName`. |
| **Status side‑effects** | Updates `$CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile` with flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_job_ran_time`, etc.) and the current script PID. |
| **Failure side‑effects** | Sends an email to `$GTPMailId` (CC `$ccList`) and sets `MNAAS_email_sdp_created=Yes` to indicate an SDP ticket has been raised. |

---

## 4. Integration Points & System Context

| Connected Component | Interaction |
|---------------------|-------------|
| **Other “move” scripts** (e.g., `MNAAS_report_data_loading.sh`, `MNAAS_non_move_files_from_staging.sh`) | Consume the Hive table populated by this script; they may read the same status file to decide when to start downstream processing. |
| **Status‑file (`*_ProcessStatusFile`)** | Shared across the entire MNAAS pipeline; scripts read/write flags to coordinate execution order and to surface health to monitoring dashboards. |
| **Cron scheduler** | Typically scheduled (e.g., nightly) to invoke this script; the PID lock prevents overlapping runs if a previous execution overruns. |
| **Alerting / ticketing system** | The email routine is a thin wrapper that also triggers an SDP ticket (the actual ticket creation is external to the script). |
| **Data Lake (HDFS/Hive/Impala)** | This script is the *source‑to‑lake* loader for the CBF GNV invoice view; downstream analytics jobs (e.g., billing reports) depend on the refreshed table. |
| **Database credentials** | Supplied via the properties file; other scripts that need the same source DB reuse the same credentials. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plain‑text DB credentials** in the properties file | Credential leakage, compliance breach | Move passwords to a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and source them at runtime. |
| **Stale PID lock** (process crashed, PID reused) | Subsequent runs blocked indefinitely | Add a sanity check on the PID’s start time (`ps -p $PID -o lstart=`) and purge the lock after a configurable timeout. |
| **Single‑mapper Sqoop import** (`-m 1`) may be a bottleneck for large tables | Long run times, missed SLA | Evaluate parallelism (`-m >1`) after testing; ensure target HDFS directory is partitioned appropriately. |
| **Hard‑coded schema/table names** | Breakage if upstream DB changes | Externalize schema/table names to the properties file; add validation step before import. |
| **No retry on transient failures** (network, HDFS, Hive) | Job failure for recoverable issues | Wrap Sqoop/Hive commands in a retry loop with exponential back‑off. |
| **Mail command may fail silently** | No alert sent, issue unnoticed | Capture mail exit status; fallback to an alternative notification channel (e.g., Slack, PagerDuty). |
| **Log file growth** (append‑only) | Disk exhaustion over time | Rotate logs via `logrotate` or implement size‑based truncation within the script. |

---

## 6. Running & Debugging the Script

| Step | Command / Action |
|------|------------------|
| **1. Verify environment** | Ensure Hadoop, Hive, Sqoop, Impala, and `mail` are in `$PATH`. |
| **2. Source properties manually (optional)** | `source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties` |
| **3. Execute** | `bash /path/to/MNAAS_sqoop_bareport.sh` (normally invoked by cron). |
| **4. Enable verbose/debug** | Uncomment the `set -x` line at the top or run `bash -x MNAAS_sqoop_bareport.sh`. |
| **5. Check logs** | Tail the log file: `tail -f $CBF_GNV_Invoice_generation_table_SqoopLogName`. |
| **6. Verify status file** | `cat $CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile` – look for `MNAAS_job_status=Success`. |
| **7. Validate Hive table** | `hive -e "SELECT COUNT(*) FROM ${dbname}.${CBF_GNV_Invoice_generation_table_name};"` |
| **8. Debug failures** | - Examine the log for the exact step that failed (Sqoop, Hive load, Impala refresh). <br> - Run the failing command manually (e.g., the Sqoop import line) to see detailed error output. |
| **9. Clean up** | If a stale PID lock exists: `sed -i 's/MNAAS_Script_Process_Id=.*/MNAAS_Script_Process_Id=0/' $CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile`. |

---

## 7. External Configuration & Files Referenced

| File / Variable | Purpose |
|-----------------|---------|
| `MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties` | Central configuration – defines DB connection (`OrgDetails_*`), HDFS directory (`CBF_GNV_Invoice_generation_table_Dir`), Hive table (`CBF_GNV_Invoice_generation_table_name`), log file (`CBF_GNV_Invoice_generation_table_SqoopLogName`), status file (`CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile`), email recipients (`GTPMailId`, `ccList`), Impala host (`IMPALAD_HOST`), script name (`MNAAS_Sqoop_CBF_GNV_Invoice_generation_Scriptname`), etc. |
| `$CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile` | Shared status/heartbeat file used by the whole MNAAS pipeline. |
| `$CBF_GNV_Invoice_generation_table_SqoopLogName` | Append‑only log file for this script’s execution. |
| `$CBF_GNV_Invoice_generation_table_Dir` | HDFS staging directory where Sqoop writes raw files. |
| `$dbname` | Hive database name (e.g., `mnaas`). |
| `$location_table_name` (used only in email body) | Likely a typo; should be `$CBF_GNV_Invoice_generation_table_name`. |

---

## 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace plain‑text `OrgDetails_Password` with a runtime fetch from a secrets manager; modify the Sqoop command to use `--password-file` or Kerberos authentication.  
2. **Robust PID & Concurrency Control** – Implement a lock file with a timestamp and automatic expiry (e.g., using `flock` or a custom timeout check) to avoid deadlocks when a previous run crashes.  

---