**High‑Level Documentation – `MNAAS_Traffic_Adhoc_Aggr_feb_mismatch.sh`**  

---

### 1. Purpose (one‑paragraph summary)  
This script orchestrates the ad‑hoc correction and aggregation of February traffic usage data for the MNAAS (Move‑N‑Analytics‑As‑a‑Service) platform. It removes stale Hive/Impala partitions, runs a Java‑based aggregation job that merges the corrected data into a temporary table, then loads the final results into the production traffic‑aggregation table. Throughout the run it updates a shared process‑status file, writes detailed logs, refreshes Impala metadata, and on failure creates an SDP ticket and notification email. The script is designed to be idempotent and safe to re‑run after a failure, resuming from the last successful step based on the status flag.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_remove_partitions_from_the_table`** | Updates status flag to *1*, checks that the partition‑drop Java job is not already running, invokes `drop_partitions_in_table` with the February‑mismatch partition list, logs success/failure, and refreshes the target Impala table. |
| **`MNAAS_traffic_aggr_adhoc_inter_table_loading`** | Sets flag to *2*, ensures the intermediate‑load Java job is not already running, runs the aggregation class (`$traffic_aggr_adhoc_aggregation_classname`) that reads the corrected CSV files and writes to a temporary Hive table, logs outcome, and refreshes the intermediate Impala view. |
| **`MNAAS_traffic_aggr_adhoc_table_loading`** | Sets flag to *3*, launches a background refresh script (`traffic_aggr_adhoc_refresh.sh`), runs the final load Java class (`$MNAAS_traffic_aggr_adhoc_load_classname`) that moves data from the temp table to the production table, refreshes Impala metadata, and logs success/failure. |
| **`terminateCron_successful_completion`** | Resets the status flag to *0* (idle), records a successful run timestamp, writes “Success” to the status file, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, calls `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Marks the job as “Failure” in the status file, checks if an SDP ticket/email has already been generated, and if not sends a formatted `mailx` message to the support mailbox and updates the status file to indicate that a ticket has been created. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Traffic_Adhoc_Aggr.properties`. Provides: <br>• Log path (`$MNAAS_traffic_aggr_adhocLogPath`) <br>• Process‑status file (`$MNAAS_traffic_aggr_adhoc_ProcessStatusFileName`) <br>• Hive/Impala connection info (`$HIVE_HOST`, `$IMPALAD_HOST`, ports) <br>• DB & table names (`$dbname`, `$traffic_aggr_adhoc_tblname`, etc.) <br>• Java class & JAR locations (`$CLASSPATHVAR`, `$MNAAS_Main_JarPath`, `$Generic_Jar_Names`) <br>• Email/SDP ticket parameters (`$SDP_ticket_from_email`, `$SDP_ticket_to_email`, `$MOVE_DEV_TEAM`). |
| **Static Input Files** | `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/MNAAS_Traffic_AdhocPartitions_Daily_Uniq_FileName_Feb_mismatch` – list of partition identifiers that must be dropped before re‑aggregation. |
| **Dynamic Input** | Data files (CSV/Parquet) referenced inside the Java aggregation class – not directly visible in the script but expected in a predefined HDFS location. |
| **Outputs** | • Updated process‑status file (flags, timestamps, job status). <br>• Log file at `$MNAAS_traffic_aggr_adhocLogPath`. <br>• Refreshed Impala tables (`$traffic_aggr_adhoc_tblname`, `$traffic_aggr_adhoc_inter_tblname`). |
| **Side Effects** | • Executes Hive/Impala DDL via Java and `impala-shell`. <br>• May spawn background process `traffic_aggr_adhoc_refresh.sh`. <br>• Sends email/SDP ticket on failure. |
| **Assumptions** | • Java runtime and required JARs are present on the host. <br>• Impala and Hive services are reachable (`$IMPALAD_HOST`, `$HIVE_HOST`). <br>• The status file is writable by the script user. <br>• No other instance of the same Java job is running (checked via `ps`). <br>• `mailx` is configured for outbound SMTP. |

---

### 4. Integration Points (how it connects to other scripts/components)  

| Component | Connection Detail |
|-----------|-------------------|
| **Cron Scheduler** | Typically invoked by a daily/weekly cron entry (e.g., `MNAAS_Traffic_Adhoc_Aggr_feb_mismatch.sh` is called from a driver script such as `MNAAS_Traffic_Adhoc_Aggr_driver.sh`). |
| **`traffic_aggr_adhoc_refresh.sh`** | Launched in background from `MNAAS_traffic_aggr_adhoc_table_loading` to perform any post‑load housekeeping (e.g., cache invalidation). |
| **Java Jobs** | • `drop_partitions_in_table` – removes stale partitions. <br>• `$traffic_aggr_adhoc_aggregation_classname` – builds the intermediate aggregation table. <br>• `$MNAAS_traffic_aggr_adhoc_load_classname` – moves data to the final table. All share the same classpath and rely on the same configuration file. |
| **Impala** | `impala-shell` commands refresh metadata for the tables after each stage. |
| **Process‑Status File** | Shared across all MNAAS traffic‑aggregation scripts (e.g., `MNAAS_Traffic_Adhoc_Aggr_apr21_mismatch.sh`). The flag values (0‑3) allow the driver to resume after a crash. |
| **Monitoring / Alerting** | Failure triggers an SDP ticket via email; the ticketing system (presumably ServiceNow or internal SDP) consumes the mail. |
| **Other Ad‑hoc Scripts** | Similar scripts for other months (e.g., `MNAAS_Traffic_Adhoc_Aggr_apr21_mismatch.sh`) use the same property file format and status‑file conventions, enabling a common orchestration layer. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent Java Job Execution** – `ps` check may miss fast‑starting duplicate processes, leading to race conditions on Hive tables. | Data corruption / duplicate loads. | Use a lock file (e.g., `flock`) or Hadoop/YARN application IDs for robust mutual exclusion. |
| **Stale Process‑Status Flag** – If the script aborts without reaching a termination function, the flag may stay at 1‑3, preventing future runs. | Permanent blockage of the pipeline. | Add a watchdog that resets flags after a configurable timeout, or store a heartbeat timestamp and validate freshness before proceeding. |
| **Impala Refresh Failure** – `impala-shell` may return non‑zero but the script does not check the exit code. | Subsequent queries see stale metadata. | Capture and evaluate the exit status; on failure, retry or abort with proper logging. |
| **Mailx / SDP Ticket Failure** – If the mail server is down, the failure may go unnoticed. | No alert to support, prolonged outage. | Queue the email (e.g., using `sendmail` or an internal ticket API) and retry; also write a flag to the status file for manual follow‑up. |
| **Hard‑coded Paths / Environment** – Paths are absolute; moving the script or changing the user home breaks execution. | Deployment errors. | Externalize base directories into the properties file and reference via variables. |
| **Insufficient Logging Rotation** – Log file grows indefinitely. | Disk exhaustion. | Implement log rotation (e.g., `logrotate`) or size‑based truncation within the script. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Prerequisite** | Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Traffic_Adhoc_Aggr.properties` is present and contains correct values. |
| **Manual Execution** | ```bash\nchmod +x MNAAS_Traffic_Adhoc_Aggr_feb_mismatch.sh\n./MNAAS_Traffic_Adhoc_Aggr_feb_mismatch.sh\n``` |
| **Enable Verbose Logging** | The script already runs with `set -x`; you can also tail the log: `tail -f $MNAAS_traffic_aggr_adhocLogPath`. |
| **Force a Specific Stage** | Edit the status file (`$MNAAS_traffic_aggr_adhoc_ProcessStatusFileName`) and set `MNAAS_Daily_ProcessStatusFlag` to 1, 2, or 3 to start from a particular step. |
| **Check Running Instances** | `ps -ef | grep MNAAS_Traffic_Adhoc_Aggr_feb_mismatch.sh` – ensure only one instance is active. |
| **Debug Java Jobs** | Java classes write to the same log path; you can increase Java logging (e.g., add `-Dlog4j.debug=true` to the `java` command). |
| **Validate Impala Refresh** | After the script finishes, run `impala-shell -i $IMPALAD_HOST -q "SHOW TABLES;"` and verify the target tables are present and have the expected partitions. |
| **Simulate Failure** | Force a non‑zero exit from one of the Java calls (e.g., rename a JAR) and verify that an SDP ticket email is generated and the status file shows `MNAAS_job_status=Failure`. |

---

### 7. External Configuration & Environment Variables  

| Variable (populated from the properties file) | Meaning / Usage |
|----------------------------------------------|-----------------|
| `MNAAS_traffic_aggr_adhocLogPath` | Path to the script’s log file (stderr redirected). |
| `MNAAS_traffic_aggr_adhoc_ProcessStatusFileName` | Shared status file that stores flags, timestamps, PID, etc. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components for the three Java jobs. |
| `Dname_MNAAS_drop_partitions_traffic_aggr_adhoc_tbl`, `Dname_MNAAS_Load_Daily_traffic_aggr_adhoc_daily_aggr_temp`, `Dname_MNAAS_traffic_aggr_adhoc_load` | Unique identifiers used to detect already‑running Java processes. |
| `drop_partitions_in_table`, `traffic_aggr_adhoc_aggregation_classname`, `MNAAS_traffic_aggr_adhoc_load_classname` | Fully‑qualified Java class names invoked by the script. |
| `traffic_aggr_adhoc_tblname_refresh`, `traffic_aggr_adhoc_inter_tblname_refresh` | Impala `REFRESH` statements (e.g., `REFRESH $traffic_aggr_adhoc_tblname`). |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Impala and Hive. |
| `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM` | Email routing for failure notifications. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java jobs). |
| `MNAAS_traffic_aggr_adhoc_ScriptName` | Usually set to the script filename; used in logs and email body. |

---

### 8. Suggested Improvements (TODO)  

1. **Replace `ps | grep` concurrency checks with a robust lock mechanism** (e.g., `flock /var/lock/mnaas_traffic_adhoc.lock`) to avoid race conditions and simplify the code.  
2. **Add explicit error handling for `impala-shell` commands** – capture the exit status, retry on transient failures, and abort with a clear log entry if the refresh does not succeed.  

--- 

*End of documentation.*