**Tableau_security_Sqoop.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`Tableau_security_Sqoop.sh` is a daily ETL driver that extracts Tableau security data from an Oracle source via Sqoop, stages the raw files in HDFS, loads them into a Hive temporary table, merges the data into the production Hive table, and refreshes the corresponding Impala view. It maintains a shared process‑status file to coordinate execution, logs all actions, and on failure (in production) raises an SDP ticket and sends an email notification. The script is intended to run under cron and is part of the broader “MNAAS” data‑move pipeline that populates reporting tables for Tableau dashboards.

---

### 2. Key Functions & Responsibilities
| Function | Responsibility |
|----------|----------------|
| **`tableau_security_Sqoop`** | Executes the full load: cleans HDFS target dir, runs Sqoop import, sets permissions, truncates Hive temp & target tables, loads HDFS files into the temp table, inserts into the final table, refreshes Impala, and logs each step. |
| **`terminateCron_successful_completion`** | Updates the shared status file to indicate success, writes final log entries, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step` when `ENV_MODE=PROD`, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Marks the job as failed in the status file, checks whether an SDP ticket/email has already been generated, and if not sends a formatted email (via `mailx`) and creates an SDP ticket. Updates the status file to record ticket creation. |
| **Main block** | Guard against concurrent runs (process‑count check), reads the process‑status flag, decides whether to start the load or skip, and finally calls the appropriate termination routine. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `Tableau_security_Sqoop.properties` – defines all `$tableau_security_*` variables, `$dbname`, `$IMPALAD_HOST`, `$ENV_MODE`, email lists, log file base name, status‑file path, etc. |
| **External services** | • Oracle DB (via JDBC URL) <br>• Hadoop HDFS (target directory) <br>• Hive Metastore (DDL/DML) <br>• Impala daemon (refresh) <br>• Local mail system (`mailx`) for SDP ticket/email <br>• SDP ticketing system (via email) |
| **Files created/modified** | • HDFS directory `${tableau_security_SqoopDir}` (raw Sqoop files) <br>• Hive tables `$dbname.$tableau_security_temp_tblname` and `$dbname.$tableau_security_tblname` (data) <br>• Log file `${tableau_security_SqoopLog}_<date>` (append) <br>• Process‑status file `${tableau_security_Sqoop_ProcessStatusFile}` (flags, timestamps) |
| **Assumptions** | • The properties file exists and is readable. <br>• Oracle credentials are valid and have SELECT on the query. <br>• Hive/Impala services are up and the target tables already exist with matching schema. <br>• The script runs under a user with Hadoop, Hive, and Impala client binaries in `$PATH`. <br>• `ps` filtering reliably identifies concurrent instances (script name is unique). |
| **Side effects** | • Deletes any existing files in the target HDFS directory before import. <br>• Truncates Hive tables (data loss if run against wrong environment). <br>• May generate SDP tickets and send emails on failure. |

---

### 4. Integration Points with Other Scripts / Components
| Connection | Description |
|------------|-------------|
| **Shared Process‑Status File** | `${tableau_security_Sqoop_ProcessStatusFile}` is used by multiple “MNAAS” scripts to coordinate daily loads (e.g., `MNAAS_Traffic_tbl_*`, `Move_Invoice_Register.sh`). The flags `MNAAS_Daily_ProcessStatusFlag` and `MNAAS_job_status` are read/written here. |
| **Common Logging Convention** | Log files follow the pattern `${tableau_security_SqoopLog}_YYYY-MM-DD`. Other scripts rotate or archive logs using the same naming scheme. |
| **Environment Variables** | `ENV_MODE` (PROD/DEV) is set globally and influences error‑handling (ticket creation). |
| **Email / SDP Ticketing** | The email address lists (`$GTPMailId`, `$MOVE_DEV_TEAM`, `$SDP_Receipient_List`) are defined centrally and reused across the suite for failure notifications. |
| **Hive/Impala Tables** | The target tables (`$dbname.$tableau_security_tblname`) are later consumed by Tableau dashboards and possibly by downstream aggregation scripts (e.g., `MNAAS_Usage_Trend_Aggr.sh`). |
| **Cron Scheduler** | The script is invoked by a daily cron entry (not shown) that sets up the environment (e.g., `source /etc/profile.d/hadoop.sh`). |

---

### 5. Operational Risks & Recommended Mitigations
| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – duplicate runs could corrupt data. | The script already checks process count; reinforce with a lock file (`flock`) and ensure the `ps` filter matches the full path. |
| **Credential exposure** – Oracle password in plain text. | Move credentials to a secure vault (e.g., Hadoop Credential Provider) and retrieve via `hadoop credential` command. |
| **Data loss on truncation** – accidental run against wrong DB/environment. | Add an explicit environment sanity check (compare `$ENV_MODE` and `$dbname` against an allow‑list) before truncating tables. |
| **Sqoop import failure** – network or source DB outage. | Implement retry logic (e.g., loop with exponential back‑off) and alert if retries exceed threshold. |
| **HDFS space exhaustion** – large import may fill HDFS. | Monitor HDFS usage before import; abort with clear log if free space < threshold. |
| **SDP ticket spam** – repeated failures generate many tickets. | Ensure the “email_sdp_created” flag is reliably reset only after a successful run; consider rate‑limiting ticket creation. |

---

### 6. Running / Debugging the Script

| Step | Command / Action |
|------|-------------------|
| **Manual execution** | `bash /path/to/Tableau_security_Sqoop.sh` (ensure the properties file is accessible). |
| **Enable verbose tracing** | Uncomment `set -x` near the top of the script to print each command as it runs. |
| **Check status before run** | `grep MNAAS_Daily_ProcessStatusFlag <status_file>` – should be `0` or `1`. |
| **View logs** | `tail -f ${tableau_security_SqoopLog}_$(date +%F)` while the job runs. |
| **Force re‑run** | Stop any existing process (`pkill -f Tableau_security_Sqoop.sh`), reset the flag in the status file to `0`, then execute. |
| **Debug Sqoop query** | Copy the `${tableau_security_Query}` from the properties file and run it directly with `sqlplus` or `sqoop eval` to verify syntax. |
| **Validate Hive load** | After successful run, run `hive -e "select count(*) from $dbname.$tableau_security_tblname;"` to confirm row count. |
| **Impala refresh check** | `impala-shell -i $IMPALAD_HOST -q "show tables like '$tableau_security_tblname';"` and query a few rows. |

---

### 7. External Configuration & Environment Variables

| Variable (from properties) | Role |
|----------------------------|------|
| `tableau_security_SqoopDir` | HDFS target directory for Sqoop files. |
| `tableau_security_SqoopLog` | Base path for daily log files. |
| `tableau_security_Sqoop_ProcessStatusFile` | Shared status/flag file. |
| `tableau_security_ServerName`, `tableau_security_PortNumber`, `tableau_security_Service` | Oracle JDBC connection details. |
| `tableau_security_Username`, `tableau_security_Password` | Oracle credentials (currently plain text). |
| `tableau_security_Query` | The SELECT statement executed by Sqoop. |
| `tableau_security_tblname`, `tableau_security_temp_tblname` | Hive target and staging tables. |
| `dbname` | Hive database name. |
| `IMPALAD_HOST` | Hostname of the Impala daemon for refresh. |
| `ENV_MODE` | Execution environment (PROD/DEV). |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `SDP_Receipient_List` | Email recipients for failure notifications. |
| `tableau_security_Sqoop_Scriptname` | Usually set to the script filename for logging. |

---

### 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace plain‑text Oracle password with a call to Hadoop Credential Provider or an external secret manager (e.g., HashiCorp Vault).  
2. **Idempotent Load Logic** – Instead of truncating the target table, implement an upsert (MERGE) or use partitioned tables with a “drop‑and‑add” strategy to avoid data loss if the script is rerun unintentionally.  

---