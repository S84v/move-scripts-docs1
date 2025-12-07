**File:** `move-mediation-scripts/bin/MNAAS_Daily_Market_Management_Report.sh`

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily “Market Management Report” (MMR) pipeline. It (a) prepares HDFS permissions, (b) runs a Spark job that extracts and transforms source data for a configurable date window, (c) refreshes an Impala summary view, (d) executes a Java‑based retention routine that drops partitions older than a configured retention period, and (e) updates a shared process‑status file while handling success/failure logging and automated email alerts. The script also guards against concurrent executions by checking a PID stored in the status file.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `load_data_into_mmr_table` | * Sets process‑status flag to *running* (value 1). <br>* Adjusts HDFS permissions on the target Hive warehouse directory. <br>* Computes `start_date`, `end_date`, `year_month` (current month). <br>* Submits the Spark job (`$MNAAS_Daily_Market_Mgmt_Report_Pyfile`) with YARN cluster mode. <br>* On Spark success, runs an Impala query (`$MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh`). <br>* Logs success/failure and triggers termination helpers. |
| `MNAAS_retention_for_market_management_report` | * Sets process‑status flag to *retention* (value 2). <br>* Re‑applies HDFS permissions (defensive). <br>* Executes a Java class (`$DropNthMonthOlderPartition`) to drop Hive partitions older than `$mnaas_retention_period_market_mgmt_report`. <br>* On success, refreshes the same Impala view; on failure, aborts with error handling. |
| `terminateCron_successful_completion` | * Resets status flags to *idle* (0) and marks job as `Success`. <br>* Writes final log entries and exits with status 0. |
| `terminateCron_Unsuccessful_completion` | * Logs failure, sends an email alert, and exits with status 1. |
| `email_on_reject_triggering_step` | * Builds a minimal RFC‑822 message and sends it via `mailx` to the SDP ticket mailbox, copying the T0 group. |
| **Main program** (bottom of file) | * Prevents parallel runs by reading the PID from the status file and checking the process table. <br>* Depending on the current flag (0, 1, 2) decides whether to run the load step, the retention step, or both. <br>* Updates the PID in the status file before launching work. |

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Configuration files** | `MNAAS_Daily_Market_Management_Report.properties` – defines all environment variables used (paths, DB names, hostnames, Spark/Java binaries, retention period, email addresses, etc.). |
| **Process‑status file** | `$MNAAS_Daily_Market_Management_Report_ProcessStatusFileName` – a simple `key=value` flat file used for inter‑script coordination, PID tracking, and status reporting. |
| **Log file** | `$MNAAS_Daily_Market_Mgmt_Report_logpath` (often a directory + filename). All `logger -s` calls append to this file. |
| **External services** | • HDFS (chmod on `/user/hive/warehouse/mnaas.db/market_management_report_sims/*`). <br>• YARN/Spark cluster (job submission). <br>• Impala daemon (`$IMPALAD_HOST`). <br>• Hive metastore (via Java class). <br>• SMTP (via `mailx`). |
| **Outputs** | • Populated Hive table `market_management_report_sims` (via Spark). <br>• Refreshed Impala view (`$MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh`). <br>• Potentially dropped old Hive partitions (retention). <br>• Updated status file and log entries. |
| **Side‑effects** | • Changes file permissions on HDFS directories (777). <br>• Sends email alerts on failure. <br>• May create an SDP ticket (implicit via email). |
| **Assumptions** | • All required environment variables are correctly exported by the sourced properties file. <br>• Spark, Impala, Hive, and Java binaries are reachable on the execution host. <br>• The status file is writable by the script user and not corrupted. <br>• `mailx` is configured to relay mail to the internal ticketing system. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | How the Interaction Occurs |
|---------------------|----------------------------|
| **Other MNAAS daily scripts** (e.g., `MNAAS_Daily_KYC_Feed_Loading.sh`) | Share the same *process‑status* file naming convention and logging directory. They may run in the same cron window; the PID guard prevents overlap. |
| **`MNAAS_Daily_Market_Management_Report.properties`** | Centralizes configuration; any change (e.g., new Spark jar path) propagates to this script and any sibling scripts that source the same file. |
| **Spark job (`*.py`)** | Invoked via `spark-submit`; the Python script contains the actual ETL logic for the MMR. |
| **Impala (`impala-shell`)** | Used to execute a refresh/refresh‑materialized‑view statement after data load and after retention. |
| **Java retention utility (`DropNthMonthOlderPartition`)** | Called with classpath and arguments to prune old Hive partitions. Likely a shared JAR used by other retention scripts. |
| **Monitoring / Alerting** | Email sent to `insdp@tatacommunications.com` with copies to `$T0_email`; the subject/category follows the internal Move‑Mediation taxonomy. |
| **Cron scheduler** | The script is expected to be launched by a daily cron entry; the PID guard ensures only one instance runs per day. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Stale PID / orphaned status file** – If the script crashes, the PID remains and subsequent runs think a process is still active. | Add a health‑check (e.g., verify the PID still exists and is the same script) before aborting; optionally clean up stale entries after a timeout. |
| **HDFS permission changes (777)** – Overly permissive ACLs may expose data. | Restrict to required user/group (`chmod 750`) and rely on Hive/Impala ACLs; audit after each run. |
| **Spark job failure not captured** – Only exit code `$?` is checked; Spark may exit 0 while the job fails internally. | Capture Spark logs and parse for “FAILED” or use `--conf spark.yarn.maxAppAttempts=1` and check the YARN application state via `yarn application -status`. |
| **Retention job data loss** – Dropping partitions incorrectly could delete needed data. | Log the partitions to be dropped before execution; add a dry‑run flag; keep a backup of the Hive metastore for the retention period. |
| **Email alert spam** – Repeated failures could flood inboxes. | Implement a throttling mechanism (e.g., only send first failure per day, then aggregate). |
| **Hard‑coded paths** – Any change in HDFS layout requires script edit. | Externalize directory paths in the properties file and reference them via variables. |

---

## 6. Running / Debugging the Script  

1. **Preparation**  
   ```bash
   # Ensure the properties file exists and is readable
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Market_Management_Report.properties
   # Verify required env vars are set (example)
   env | grep MNAAS_Daily_Market_Management_Report
   ```

2. **Manual Execution** (for a specific date range)  
   ```bash
   export setparameter="setparameter"   # placeholder if required by upstream logic
   ./MNAAS_Daily_Market_Management_Report.sh
   ```

3. **Debug Mode**  
   - The script already runs with `set -x` (trace). To increase verbosity, add `export VERBOSE=1` and insert `[[ -n $VERBOSE ]] && echo "debug: $var"` at strategic points.  
   - Check the generated log file (`$MNAAS_Daily_Market_Mgmt_Report_logpath`) for timestamps and error messages.  
   - Verify Spark job status via YARN UI (`http://<resource_manager>:8088`) using the application ID printed in the log.  

4. **Common Failure Checks**  
   - **PID conflict:** `ps -p <PID>` – if the process is dead, remove the PID line from the status file.  
   - **Spark exit code:** Look for `spark-submit` return code in the log.  
   - **Impala errors:** Impala shell returns are not captured; add `|| { logger -s "Impala failed"; exit 1; }` after each `impala-shell` call for stricter handling.  
   - **Java retention errors:** Examine the Java class logs (they are redirected to `${MNAAS_Daily_Market_Mgmt_Report_logpath}$(date +_%F)`).  

5. **Post‑run validation**  
   ```bash
   # Verify Hive table row count
   hive -e "SELECT COUNT(*) FROM mnaas.market_management_report_sims;"
   # Verify latest partition exists
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/market_management_report_sims/
   ```

---

## 7. External Configurations & Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_Daily_Market_Management_Report_ProcessStatusFileName` | Path to the shared status file. |
| `MNAAS_Daily_Market_Mgmt_Report_logpath` | Base path for log files (often a directory). |
| `MNAAS_Daily_Market_Mgmt_Report_Pyfile` | Full path to the Spark Python ETL script. |
| `IMPALAD_HOST` | Hostname of the Impala daemon for `impala-shell`. |
| `MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh` | SQL statement (or view refresh command) executed after load/retention. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath components for the retention utility. |
| `DropNthMonthOlderPartition` | Fully‑qualified Java class name that performs partition dropping. |
| `dbname`, `MNAAS_Daily_Market_Mgmt_Report_tblname` | Hive database and table targeted by retention. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive metastore connection details used by the Java utility. |
| `mnaas_retention_period_market_mgmt_report` | Retention period (e.g., “6” for six months). |
| `MNAAS_Daily_Market_Mgmt_Report_Script` | Human‑readable script identifier used in logs/emails. |
| `SDP_ticket_from_email`, `T0_email` | Email addresses for ticket creation and CC. |
| `setparameter` | A placeholder evaluated early; may contain additional `export` statements. |

If any of these are missing or malformed, the script will abort early (often with a `logger` entry). Verify the properties file before scheduling.

---

## 8. Suggested Improvements (TODO)

1. **Robust PID & Status Management** – Replace the ad‑hoc PID check with a lock file (`flock`) or a more reliable coordination service (e.g., Zookeeper) to avoid stale PID issues and race conditions.
2. **Enhanced Error Propagation from Impala & Spark** – Capture and log the full stdout/stderr of `impala-shell` and `spark-submit`, and exit with non‑zero status if the Impala query returns an error code. This will prevent silent failures when the Spark job succeeds but the downstream view refresh fails.  

---