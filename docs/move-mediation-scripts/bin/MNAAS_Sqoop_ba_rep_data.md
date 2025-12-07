**MNAAS_Sqoop_ba_rep_data.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Sqoop_ba_rep_data.sh` is a monthly batch job that extracts billing‑account‑report data from an Oracle source system via Sqoop, writes the raw delimited files to a staging HDFS directory, and loads them into a partitioned Hive table (`ba_rep_data`) for the previous calendar month. After a successful Hive load the script refreshes the corresponding Impala table. The job maintains a shared process‑status file to coordinate execution, logs all actions, and on failure sends an email alert and creates an SDP ticket.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`ba_rep_data_table_Sqoop`** | Core workflow: set status flags, clean HDFS staging area, compute previous month, drop existing Hive partition, run Sqoop import, load data into Hive partition, refresh Impala, log each step. |
| **`terminateCron_successful_completion`** | Reset status flags to *Success*, record run time, write final log entries, exit 0. |
| **`terminateCron_Unsuccessful_completion`** | Log failure, update run‑time flag, invoke email/SDP alert, exit 1. |
| **`email_and_SDP_ticket_triggering_step`** | If not already done, send a formatted email to the operations team (and CC list) and set the “email sent” flag in the status file. |
| **Main program block** | Guard against concurrent runs using PID stored in the status file, decide whether to start the job based on the daily‑process flag, and invoke the core function. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Sqoop_ba_rep_data.properties` – defines all runtime variables (DB credentials, HDFS paths, Hive table names, log file names, email recipients, etc.). |
| **Environment variables** | None required beyond those defined in the properties file; script uses `set -x` for debugging. |
| **External services** | • Oracle DB (via JDBC URL) <br>• Hadoop HDFS <br>• Hive Metastore <br>• Impala daemon (`IMPALAD_HOST`) <br>• Local mail subsystem (`mail` command) |
| **Files** | • Process‑status file (`$ba_rep_data_Sqoop_ProcessStatusFile`) – shared across scripts. <br>• Log file (`$ba_rep_data_table_SqoopLogName`). <br>• HDFS staging directory (`$ba_rep_data_table_Dir`). |
| **Outputs** | • Partitioned Hive table `dbname.ba_rep_data` (new partition `bill_month=YYYY‑MM`). <br>• Updated process‑status file (flags, run time, job status). <br>• Log entries (stdout & syslog). <br>• Optional email alert on failure. |
| **Side effects** | • Deletes any existing files under the staging HDFS directory before each run. <br>• May create a new SDP ticket (via external ticketing system – not shown in script). |
| **Assumptions** | • The properties file exists and contains valid values. <br>• Oracle credentials have SELECT rights on the source tables. <br>• Hive table `ba_rep_data` is already created with a `bill_month` partition column. <br>• The process‑status file is writable by the script user. <br>• The mail command is configured and can reach the recipients. |

---

### 4. Interaction with Other Components

| Component | How this script connects |
|-----------|--------------------------|
| **Other MNAAS batch scripts** | All scripts share the same *process‑status* file pattern (`MNAAS_…_ProcessStatusFile`). They read/write flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, etc.) to coordinate daily/weekly/monthly pipelines. |
| **Hive/Impala tables** | Downstream analytics jobs query `dbname.ba_rep_data`. Upstream scripts may populate reference tables used by the Sqoop query (`$ba_rep_data_table_Query`). |
| **SDP ticketing system** | The `email_and_SDP_ticket_triggering_step` function marks a ticket as created via the `MNAAS_email_sdp_created` flag; the actual ticket creation is external to the script (likely a wrapper around the email). |
| **Monitoring/Orchestration** | The script’s log file and status flags are typically consumed by a scheduler (e.g., cron, Oozie, Airflow) to decide whether to trigger the next step in the monthly data‑load chain. |
| **Properties file** | The `.properties` file is also used by sibling scripts (e.g., `MNAAS_Sqoop.sh`, `MNAAS_Sqoop_Cbf_GNV_Invoice_generation.sh`) to keep connection strings and directory names consistent. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID in status file** – if the script crashes, the PID may remain, blocking future runs. | Job never starts. | Add a sanity check on process age (e.g., `ps -p $PID -o etime=`) and clear PID if older than a threshold. |
| **Sqoop import failure** (network, auth, query error). | No data loaded; downstream jobs fail. | Validate Oracle connectivity before import; capture Sqoop exit code; retry logic or alert on first failure. |
| **Hive partition drop/load race** – concurrent jobs could drop the same partition. | Data loss or duplicate loads. | Enforce exclusive lock via the shared status file; ensure only one monthly loader runs at a time (already partially done). |
| **Permission errors on HDFS directory** (chmod 777 may be insufficient on secure clusters). | Load fails, logs incomplete. | Use proper HDFS ACLs; avoid wide open permissions; log the result of `chmod`. |
| **Email delivery failure** (mail command mis‑configured). | Operators not notified of failures. | Verify mail service health; fallback to syslog or push notification; record email send status in logs. |
| **Hard‑coded single mapper (`-m 1`)** – may be a performance bottleneck. | Long run time for large data sets. | Parameterize mapper count in properties; benchmark and adjust. |
| **Missing/incorrect properties** – script aborts silently. | No run, no clear error. | Add validation block after sourcing properties (check required vars, exit with clear message). |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the user has read access to `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_ba_rep_data.properties`.  
   - Verify that the Oracle JDBC driver is on the classpath (default Sqoop location).  
   - Confirm HDFS, Hive, Impala, and mail services are reachable.

2. **Execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts/bin
   ./MNAAS_Sqoop_ba_rep_data.sh
   ```
   - The script runs with `set -x`; all commands are echoed to the log file defined by `$ba_rep_data_table_SqoopLogName`.

3. **Checking Status**  
   - Review the process‑status file (`$ba_rep_data_Sqoop_ProcessStatusFile`) for flags `MNAAS_job_status`, `MNAAS_job_ran_time`, etc.  
   - Tail the log file for real‑time progress:
     ```bash
     tail -f $ba_rep_data_table_SqoopLogName
     ```

4. **Debugging Tips**  
   - To isolate a step, comment out later sections and re‑run.  
   - Manually run the Sqoop command extracted from the log to verify connectivity.  
   - Use `hdfs dfs -ls $ba_rep_data_table_Dir` before and after the run to confirm file creation.  
   - Verify Hive partition existence:
     ```bash
     hive -e "show partitions $dbname.$ba_rep_data_table_name;"
     ```

5. **Force Re‑run**  
   - If a previous run left the flag `MNAAS_Daily_ProcessStatusFlag=1`, reset it to `0` in the status file before re‑executing.

---

### 7. External Configuration & Environment Variables

| Variable (from properties) | Role |
|----------------------------|------|
| `ba_rep_data_table_SqoopLogName` | Full path to the script’s log file. |
| `ba_rep_data_Sqoop_ProcessStatusFile` | Shared status/flag file used for concurrency control and reporting. |
| `dbname` | Hive database name. |
| `ba_rep_data_table_name` | Hive table name (`ba_rep_data`). |
| `ba_rep_data_table_Dir` | HDFS staging directory for Sqoop output. |
| `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password` | Oracle JDBC connection details. |
| `ba_rep_data_table_Query` | Sqoop `--query` string (must end with `AND \$CONDITIONS`). |
| `IMPALAD_HOST` | Impala daemon host for `impala-shell` refresh. |
| `ccList`, `GTPMailId` | Email recipients for failure alerts. |
| `MNAAS_Sqoop_ba_rep_data_Scriptname` | Human‑readable script identifier used in logs/emails. |

If any of these are missing or empty, the script will likely fail early; adding a validation block after sourcing the properties file is recommended.

---

### 8. Suggested Improvements (TODO)

1. **Add Property Validation** – Immediately after sourcing the `.properties` file, loop through a list of required variables and abort with a clear error if any are undefined or empty.
2. **Parameterize Mapper Count** – Introduce `ba_rep_data_sqoop_mappers` in the properties file (default to 1) and pass `-m $ba_rep_data_sqoop_mappers` to Sqoop, allowing performance tuning without code changes.  

---