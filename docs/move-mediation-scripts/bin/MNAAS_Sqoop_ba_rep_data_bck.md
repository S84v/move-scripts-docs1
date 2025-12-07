**MNAAS_Sqoop_ba_rep_data_bck.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Sqoop_ba_rep_data_bck.sh` is a nightly batch job that extracts billing‑report data (`ba_rep_data`) from an Oracle source system via Sqoop, loads it into a Hive table partitioned by the previous month, and refreshes the corresponding Impala view. The script orchestrates the full data‑move lifecycle: it clears the target HDFS directory, drops any existing Hive partition, runs the Sqoop import, loads the data into Hive, refreshes Impala, and updates a shared process‑status file. It also implements single‑instance locking, detailed logging, and failure notification (email + SDP ticket) to ensure operational visibility and prevent overlapping runs.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`ba_rep_data_table_Sqoop`** | Core ETL routine: updates status flag, logs start, removes stale HDFS files, drops previous Hive partition, runs Sqoop import, loads data into Hive partition, refreshes Impala, logs success/failure. |
| **`terminateCron_successful_completion`** | Marks the process as completed successfully in the status file, logs termination, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, timestamps the status file, triggers email/SDP alert, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Sends a failure notification email (and creates an SDP ticket flag) only once per run, updates the status file accordingly. |
| **Main program block** | Implements single‑process lock using PID stored in the status file, checks the daily flag, invokes the ETL routine, and routes to the appropriate termination function. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Sqoop_ba_rep_data.properties` – defines all environment variables used (see section 5). |
| **External Services** | • Oracle DB (JDBC URL built from `OrgDetails_*` variables) <br>• Hadoop HDFS (`hadoop fs`) <br>• Hive Metastore (`hive -e`) <br>• Impala (`impala-shell`) <br>• Mail server (`mail` command) |
| **Files** | • `$ba_rep_data_table_SqoopLogName` – append‑only log file <br>• `$ba_rep_data_Sqoop_ProcessStatusFile` – plain‑text key/value status file updated throughout the run |
| **Data Flow** | Input: Oracle query defined in `$ba_rep_data_table_Query` <br>Output: Hive table `$dbname.$ba_rep_data_table_name` partition `bill_month='${prev_month}'` populated with CSV files under `$ba_rep_data_table_Dir` |
| **Side Effects** | • Deletes all files under the target HDFS directory before import <br>• Drops existing Hive partition for the previous month <br>• Updates status flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, etc.) <br>• Sends email on failure and sets `MNAAS_email_sdp_created=Yes` |
| **Assumptions** | • All required variables are correctly defined in the properties file <br>• The script runs on a node with Hadoop, Hive, Sqoop, Impala client, and `mail` installed <br>• Network connectivity to Oracle and the Hadoop cluster is stable <br>• Only one instance of the script runs at a time (PID lock works) |

---

### 4. Integration Points (how it connects to other scripts/components)

| Connection | Description |
|------------|-------------|
| **Status File (`$ba_rep_data_Sqoop_ProcessStatusFile`)** | Shared with other MNAAS batch jobs (e.g., `MNAAS_Sqoop_ba_rep_data.sh`). The same flag keys (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, etc.) are used across jobs to coordinate sequencing and monitor health. |
| **Properties File** | The sibling script `MNAAS_Sqoop_ba_rep_data.sh` sources the *same* `.properties` file, ensuring consistent DB credentials, table names, and directory paths. |
| **Hive Table** | Populated by this script; downstream reporting jobs (e.g., monthly SIM status, invoicing scripts) read from `$dbname.$ba_rep_data_table_name`. |
| **Impala Refresh** | Guarantees that any Impala‑based analytics jobs see the newly loaded data immediately. |
| **Email/SDP Alerting** | Uses the same mailing list (`$GTPMailId`, `$ccList`) and SDP ticket flag as other MNAAS scripts, providing a unified incident‑response channel. |
| **Cron Scheduler** | Typically invoked by a daily cron entry (e.g., `0 2 * * * /path/MNAAS_Sqoop_ba_rep_data_bck.sh`). The PID lock prevents overlapping runs with the non‑backup version (`MNAAS_Sqoop_ba_rep_data.sh`). |

---

### 5. External Configuration & Environment Variables

| Variable (populated in `.properties`) | Role |
|---------------------------------------|------|
| `ba_rep_data_table_SqoopLogName` | Path to the log file where all stdout/stderr are appended. |
| `ba_rep_data_Sqoop_ProcessStatusFile` | Central status file updated throughout the run. |
| `dbname` | Hive database name. |
| `ba_rep_data_table_name` | Target Hive table name. |
| `ba_rep_data_table_Dir` | HDFS directory used as Sqoop target (`--target-dir`). |
| `prev_month` | Partition value (YYYYMM) for the previous month. |
| `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password` | Oracle JDBC connection details. |
| `ba_rep_data_table_Query` | Sqoop `--query` string (must end with `AND \$CONDITIONS`). |
| `IMPALAD_HOST` | Hostname of the Impala daemon for `impala-shell`. |
| `MNAAS_Sqoop_ba_rep_data_Scriptname` | Human‑readable script identifier used in logs/emails. |
| `ccList`, `GTPMailId` | Email recipients for failure notifications. |
| `MNAAS_FlagValue` (derived) | Controls whether the job may start (0 or 1). |

*If any of these variables are missing or malformed, the script will abort with a failure logged and an email sent.*

---

### 6. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Plain‑text credentials in properties file** | Store credentials in a secure vault (e.g., Hadoop KMS, HashiCorp Vault) and source them at runtime; restrict file permissions to the service account. |
| **Single‑process lock race condition** | Ensure the status file is on a reliable shared filesystem; consider using `flock` or a Zookeeper lock for stronger guarantees. |
| **Data loss on HDFS delete before successful import** | Add a safety check (e.g., move to a temporary “staging” directory) and retain a backup of the previous successful load. |
| **Sqoop runs with a single mapper (`-m 1`)** – may be a performance bottleneck | Evaluate data volume; increase mapper count and adjust `--split-by` column if throughput is insufficient. |
| **No retry on transient failures (network, Oracle, HDFS)** | Wrap Sqoop and Hive commands in a retry loop with exponential back‑off; log each attempt. |
| **Unbounded log file growth** | Rotate logs via `logrotate` or implement size‑based truncation within the script. |
| **Email flood on repeated failures** | The script already guards against duplicate SDP tickets; ensure the email flag is persisted across runs and consider a throttling mechanism. |

---

### 7. Running & Debugging the Script

1. **Prerequisites**  
   - Verify that the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_ba_rep_data.properties` exists and is readable.  
   - Ensure the executing user has permissions on HDFS, Hive, Impala, and the status/log files.  

2. **Manual Execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_Sqoop_ba_rep_data_bck.sh
   ```
   - The script will append to the log file defined by `ba_rep_data_table_SqoopLogName`.  
   - Check the log for lines prefixed with the script name and timestamps.  

3. **Debug Mode**  
   - Uncomment the first line (`set -x`) to enable Bash trace output.  
   - Alternatively, run with `bash -x MNAAS_Sqoop_ba_rep_data_bck.sh` to see each command as it executes.  

4. **Validate Output**  
   - After successful run, verify Hive partition:  
     ```bash
     hive -e "SHOW PARTITIONS ${dbname}.${ba_rep_data_table_name};"
     ```  
   - Confirm data in Impala:  
     ```bash
     impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM ${dbname}.${ba_rep_data_table_name} WHERE bill_month='${prev_month}';"
     ```  

5. **Failure Investigation**  
   - Review the log file for the exact point of failure.  
   - Check the status file for `MNAAS_job_status=Failure` and the timestamp.  
   - Verify Oracle connectivity (`tnsping`, `sqlplus`) and HDFS health (`hdfs dfs -ls $ba_rep_data_table_Dir`).  

---

### 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace plain‑text Oracle credentials with a key‑based retrieval mechanism (e.g., `kinit` + Kerberos, or a vault API) and remove `OrgDetails_Password` from the properties file.  
2. **Add Retry Logic** – Implement a generic retry wrapper for Sqoop, Hive, and Impala commands with configurable max attempts and back‑off intervals to improve resilience against transient infrastructure glitches.  

---