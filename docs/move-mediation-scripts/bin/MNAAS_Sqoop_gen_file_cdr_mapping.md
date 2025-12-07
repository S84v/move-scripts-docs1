**MNAAS_Sqoop_gen_file_cdr_mapping.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`MNAAS_Sqoop_gen_file_cdr_mapping.sh` is a daily cron‑driven ETL driver that extracts the *gen_file_cdr_mapping* data set from an Oracle source, lands the raw rows into a partitioned Hive table (partitioned by the previous month’s `bill_month`), and refreshes the corresponding Impala metadata. It maintains a lightweight process‑status file to prevent overlapping runs, logs all actions, and on failure raises an SDP ticket and sends an email alert.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **gen_file_cdr_mapping_table_Sqoop** | Executes the full load: updates status flag, cleans target HDFS directory, drops the previous month’s Hive partition, runs a single‑mapper Sqoop import, sets HDFS permissions, loads data into Hive partition, refreshes Impala, and logs success/failure. |
| **terminateCron_successful_completion** | Writes *Success* status, clears the running‑process flag, timestamps the run, logs completion, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, timestamps the run, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Marks the job as *Failure* in the status file, checks whether an SDP ticket/email has already been generated, and if not sends a pre‑formatted alert email (CC list configurable) and flips the `MNAAS_email_sdp_created` flag. |
| **Main program block** | Prevents concurrent executions by checking the PID stored in the status file, updates the PID if free, validates the daily‑process flag, and dispatches the load or terminates gracefully. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (external file)** | `MNAAS_Property_Files/MNAAS_Sqoop_gen_file_cdr_mapping.properties` – defines all environment‑specific variables (DB credentials, HDFS paths, Hive/Impala names, log file, status file, email recipients, etc.). |
| **Environment variables** | None required beyond those sourced from the properties file. |
| **Oracle source** | JDBC connection (`$OrgDetails_ServerName:$OrgDetails_PortNumber/$OrgDetails_Service`) with `$OrgDetails_Username` / `$OrgDetails_Password`. |
| **Hive target** | Partitioned table `$dbname.$gen_file_cdr_mapping_table_name` (partition column `bill_month`). |
| **HDFS target directory** | `$gen_file_cdr_mapping_table_Dir` – cleared before each run. |
| **Impala host** | `$IMPALAD_HOST` – used for metadata refresh. |
| **Log file** | `$gen_file_cdr_mapping_table_SqoopLogName` (appended). |
| **Process‑status file** | `$gen_file_cdr_mapping_Sqoop_ProcessStatusFileName` – holds flags, PID, timestamps, email‑sent flag, etc. |
| **Outputs** | - Populated Hive partition for the previous month.<br>- Updated status file (Success/Failure, timestamps, flags).<br>- Log entries.<br>- Optional alert email / SDP ticket on failure. |
| **Side effects** | - Deletes any existing files under the target HDFS directory.<br>- Drops the previous month’s Hive partition (if present).<br>- Changes HDFS permissions to 777 on the target directory.<br>- Triggers Impala metadata refresh. |

---

### 4. Integration Points (how it connects to other scripts/components)

| Component | Connection Detail |
|-----------|-------------------|
| **Other MNAAS Sqoop scripts** | Share the same *process‑status* file naming convention (`*_Sqoop_ProcessStatusFileName`) and logging pattern; they likely run sequentially (e.g., `MNAAS_Sqoop_ba_rep_data.sh`). |
| **Daily orchestration layer** | Invoked by a cron entry (once per day). The same cron schedule may also trigger downstream consumers that read the newly loaded Hive partition. |
| **SDP ticketing / alerting system** | Uses the `mail` command to send alerts; the `email_and_SDP_ticket_triggering_step` function is a common hook used across the MNAAS suite. |
| **Hive/Impala ecosystem** | After load, the script issues `hive -e "load data …"` and `impala-shell -i $IMPALAD_HOST -q "refresh …"` – downstream analytics jobs (e.g., reporting, billing) depend on the refreshed table. |
| **Oracle source system** | The script is the only consumer of the `gen_file_cdr_mapping` view/table defined by `$gen_file_cdr_mapping_table_Query`. Any schema change upstream must be reflected in the properties file. |
| **Status‑monitoring dashboards** | The status file fields (`MNAAS_job_status`, `MNAAS_job_ran_time`, etc.) are likely read by monitoring scripts or UI dashboards to display job health. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Hard‑coded single mapper (`-m 1`)** – may cause long runtimes or timeouts on large data volumes. | Evaluate data size; increase parallelism (`-m N`) after testing, and adjust `--target-dir` to a staging area if needed. |
| **Credentials stored in plain text** within the properties file. | Move Oracle credentials to a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and source them at runtime. |
| **Stale PID / status file** – if the script crashes, the PID may remain, preventing future runs. | Add a sanity check on PID age (e.g., if > 24 h, clear it) or use a lockfile with `flock`. |
| **Partition drop/load for previous month only** – fails on month‑rollover if the script runs after the first day of the new month. | Compute target partition based on the *run date* (e.g., `${run_date%-*}`) or add a fallback to handle the first day of the month. |
| **Log file growth** – unbounded appends may fill disk. | Implement log rotation (e.g., `logrotate`) and/or size‑based truncation within the script. |
| **No explicit error handling for Sqoop failure** – script proceeds to `chmod` even if import failed. | Capture Sqoop exit code; abort load step if non‑zero, and jump directly to failure handling. |
| **Email flood on repeated failures** – repeated runs could generate many identical alerts. | Add a throttling mechanism (e.g., only send email if last alert > X hours ago) or integrate with an incident‑management API that deduplicates tickets. |

---

### 6. Typical Execution / Debugging Steps

1. **Check prerequisites**  
   - Verify that the properties file exists and contains valid values.  
   - Ensure the Oracle JDBC driver is on the classpath (`sqoop` default).  
   - Confirm HDFS, Hive, and Impala services are reachable.

2. **Run manually (for debugging)**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Sqoop_gen_file_cdr_mapping
   ./MNAAS_Sqoop_gen_file_cdr_mapping.sh
   ```
   - The script already enables `set -x` for command tracing.  
   - Tail the log file defined by `$gen_file_cdr_mapping_table_SqoopLogName` to follow progress.

3. **Validate output**  
   - After successful run, check Hive:  
     ```bash
     hive -e "show partitions $dbname.$gen_file_cdr_mapping_table_name;"
     ```
   - Verify data count:  
     ```bash
     hive -e "select count(*) from $dbname.$gen_file_cdr_mapping_table_name where bill_month='${prev_month}';"
     ```
   - Confirm Impala sees the data: `impala-shell -i $IMPALAD_HOST -q "select count(*) from $dbname.$gen_file_cdr_mapping_table_name where bill_month='${prev_month}';"`

4. **Investigate failures**  
   - Look for “sqoop process failed” or “table load failed” entries in the log.  
   - Check the exit code of the `sqoop import` command (`echo $?`).  
   - Review Oracle side logs for connectivity or query errors.  
   - Ensure the status file flags are reset (`MNAAS_Daily_ProcessStatusFlag=0`) before re‑running.

5. **Cron verification**  
   - Confirm the cron entry (e.g., `crontab -l | grep gen_file_cdr_mapping`).  
   - Ensure the user running cron has permission to write to HDFS, Hive, and the log/status files.

---

### 7. External Configuration & Environment Dependencies

| Item | Description | Usage |
|------|-------------|-------|
| `MNAAS_Sqoop_gen_file_cdr_mapping.properties` | Centralised key‑value file sourced at script start. Contains DB connection details, HDFS paths, Hive table names, log & status file locations, email recipients, etc. | All runtime variables (`$dbname`, `$gen_file_cdr_mapping_table_Dir`, `$OrgDetails_*`, `$IMPALAD_HOST`, `$ccList`, `$GTPMailId`, …). |
| `MNAAS_Script_Process_Id` (in status file) | Stores the PID of the last successful start. | Used to detect concurrent executions. |
| `mail` command | System MTA for sending alerts. | Triggered only on failure when `MNAAS_email_sdp_created=No`. |
| `sqoop`, `hadoop`, `hive`, `impala-shell` binaries | Standard Hadoop ecosystem utilities. | Core ETL operations. |
| `logger` | Syslog logger (writes to `/var/log/messages` and the script‑specific log). | Provides timestamped audit trail. |

---

### 8. Suggested Improvements (TODO)

1. **Add robust error handling for the Sqoop step** – capture the Sqoop exit status, and if non‑zero, skip the subsequent `chmod`/Hive load and jump directly to `terminateCron_Unsuccessful_completion`.  
2. **Externalise sensitive credentials** – replace plain‑text Oracle username/password in the properties file with a call to a secret‑management service, and inject them at runtime (e.g., via environment variables or a secure token file).  

---