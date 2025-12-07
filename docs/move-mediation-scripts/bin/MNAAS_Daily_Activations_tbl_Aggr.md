**MNAAS_Daily_Activations_tbl_Aggr.sh – High‑Level Documentation**  

---

### 1. Purpose (one‑paragraph summary)  
This Bash script orchestrates the nightly aggregation of the *MNAAS Daily Activations* data set. It loads raw activation records into an aggregation table, builds daily business‑trend aggregates, and applies retention policies to both the activation and business‑trend tables. The script updates a shared process‑status file, refreshes Impala metadata after each successful step, and on failure generates an email alert and an SDP ticket. It is designed to run once per day via cron, guarding against concurrent executions.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_aggr_table** | Loads raw activation data into the daily aggregation table using a Java class (`$Insert_Daily_Aggr_table`). Updates process‑status flag to **1**, ensures only one instance runs, truncates the temporary trend file, and refreshes the Impala table. |
| **MNAAS_business_trend_aggr_table** | Generates per‑date business‑trend aggregates. Sets flag **2**, sorts the trend‑date list, iterates over each date invoking an external shell script (`$MNAASbusinessTrendAggrScriptName`), and refreshes the Impala table after each successful date. |
| **MNAAS_retention_for_activations_aggr_daily** | Applies retention (drop‑old‑partitions) to the activation aggregation table via Java class (`$DropNthOlderPartition`). Sets flag **3** and refreshes Impala metadata. |
| **MNAAS_retention_for_business_trend_daily_aggr** | Same as above but for the business‑trend aggregation table. Sets flag **4**. |
| **terminateCron_successful_completion** | Writes a *Success* status, resets the flag to **0**, records run‑time, logs completion, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email and creates an SDP ticket (via `mailx`). Updates the status file to indicate that an alert has already been sent. |
| **Main block** | Prevents concurrent runs by checking the PID stored in the status file, reads the current flag, and executes the required subset of the above functions in the correct order. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_Daily_Activations_tbl_Aggr.properties` (defines all environment variables). <br>• Control files: `$MNAAS_Daily_Activations_Aggr_ProcessStatusFileName`, `$MNAAS_Daily_Activations_AggrCntrlFileName`. <br>• Temporary data files: `$MNAAS_business_Trend_Data_filename`, `$MNAAS_business_Trend_Data_filename_sort`. |
| **Outputs** | • Aggregation tables in Hive/Impala (`$aggr_activations_daily_tblname`, `$business_trend_daily_aggr_tblname`). <br>• Updated process‑status file (flags, PID, timestamps, job status). <br>• Log file at `$MNAAS_DailyActivationsAggrLogPath`. |
| **Side Effects** | • Executes Java processes (requires Java runtime, classpath). <br>• Runs `impala-shell` to refresh table metadata. <br>• Sends email and creates SDP ticket on failure. <br>• May truncate/overwrite temporary trend files. |
| **Assumptions** | • All variables referenced in the properties file are correctly defined. <br>• Impala, Hive, and the underlying Hadoop cluster are reachable (`$IMPALAD_HOST`, `$HIVE_HOST`). <br>• Mail subsystem (`mail`, `mailx`) is functional and the SDP ticket address is valid. <br>• The external trend‑aggregation script (`$MNAASbusinessTrendAggrScriptName`) exists and is executable. |

---

### 4. Integration Points (how this script connects to other components)  

| Connection | Description |
|------------|-------------|
| **Properties file** (`MNAAS_Daily_Activations_tbl_Aggr.properties`) | Centralised configuration shared across the *MNAAS* suite (paths, class names, DB credentials, Impala queries). |
| **Java classes** (`$Insert_Daily_Aggr_table`, `$DropNthOlderPartition`) | Packaged in `$MNAAS_Main_JarPath` / `$Generic_Jar_Names`; also used by other aggregation scripts (e.g., `MNAAS_Billing_tables_loading.sh`). |
| **Business‑trend script** (`$MNAASbusinessTrendAggrScriptName`) | Called for each event date; likely resides in the same `bin/` directory and is used by other daily trend jobs. |
| **Process‑status file** (`$MNAAS_Daily_Activations_Aggr_ProcessStatusFileName`) | Shared with any script that may need to resume or skip steps (e.g., manual rerun, monitoring tools). |
| **Impala refresh queries** (`$aggr_activations_daily_tblname_refresh`, `$business_trend_daily_aggr_tblname_refresh`) | Defined in the properties file; other downstream reporting jobs depend on the refreshed metadata. |
| **SDP ticketing / email** | Failure handling integrates with the corporate incident‑management system (SDP) and the Move development mailing list (`$MOVE_DEV_TEAM`). |
| **Cron scheduler** | The script is intended to be invoked by a daily cron entry; other scripts in the suite follow the same pattern, forming a pipeline of nightly data‑move jobs. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (stale PID, duplicate runs) | Data corruption, duplicate loads | Ensure the PID file is cleared on abnormal termination; add a lockfile with `flock` as a secondary guard. |
| **Missing or malformed properties** | Script aborts early, no alerts | Validate required variables at start of script; fail fast with clear log message. |
| **Java class failure** (OOM, classpath issues) | Partial data load, retention not applied | Monitor Java exit codes; increase JVM heap if logs show OOM; keep a health‑check on the JAR versions. |
| **Impala refresh failure** | Downstream reports see stale data | Capture `impala-shell` error output; retry a limited number of times before aborting. |
| **Disk space exhaustion** (log or temporary files) | Truncate commands may fail, script stops | Rotate logs daily, monitor `/tmp` and the directory holding trend files; alert on low space. |
| **Email/SDP ticket flood** (multiple failures in same day) | Spam, ticket overload | Guard the “email already sent” flag (already present) and add a daily limit on ticket creation. |
| **External trend‑aggregation script failure** | Business‑trend table incomplete | Log the failing date, continue with next date, and raise a warning after loop finishes. |

---

### 6. Execution & Debugging Guide  

1. **Standard run (cron)**  
   ```bash
   /app/hadoop_users/MNAAS/bin/MNAAS_Daily_Activations_tbl_Aggr.sh
   ```
   - Ensure the properties file is present at the path referenced in the script.  
   - Verify that the log file `$MNAAS_DailyActivationsAggrLogPath` is writable.

2. **Manual run (for testing / debugging)**  
   ```bash
   export MNAAS_DailyActivationsAggrLogPath=/tmp/MNAAS_DailyActivationsAggr.log
   ./MNAAS_Daily_Activations_tbl_Aggr.sh
   ```
   - Tail the log: `tail -f $MNAAS_DailyActivationsAggrLogPath`.  
   - If the script exits with code 1, inspect the log for the failure point and the generated SDP ticket email.

3. **Checking current state**  
   - Process‑status file: `cat $MNAAS_Daily_Activations_Aggr_ProcessStatusFileName`.  
   - Running PID: `ps -p $(grep MNAAS_Script_Process_Id $MNAAS_Daily_Activations_Aggr_ProcessStatusFileName | cut -d= -f2)`.

4. **Debugging a specific step**  
   - To isolate the Java load step, run the Java command directly (copy from the log) and add `-Xlog:gc` or other JVM flags as needed.  
   - To test the business‑trend script, execute: `sh $MNAASbusinessTrendAggrScriptName 2024-10-01`.

5. **Log rotation**  
   - Set up a daily logrotate rule for `$MNAAS_DailyActivationsAggrLogPath` to avoid unbounded growth.

---

### 7. External Configuration & Environment Variables  

| Variable (defined in properties) | Role |
|----------------------------------|------|
| `MNAAS_DailyActivationsAggrLogPath` | Path to the script’s log file. |
| `MNAAS_Daily_Activations_Aggr_ProcessStatusFileName` | Shared status/flag file. |
| `MNAAS_Daily_Activations_AggrCntrlFileName` | Control file passed to Java loader. |
| `Dname_MNAAS_Insert_Daily_Aggr_activations_tbl` | Name used to detect running Java loader. |
| `Insert_Daily_Aggr_table` | Fully‑qualified Java class for loading activations. |
| `DropNthOlderPartition` | Java class that drops old Hive partitions. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `Generic_Jar_Names` | Java classpath / JAR locations. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Impala/Hive. |
| `aggr_activations_daily_tblname_refresh`, `business_trend_daily_aggr_tblname_refresh` | Impala `REFRESH` statements. |
| `MNAASbusinessTrendAggrScriptName` | External shell script for per‑date trend aggregation. |
| `MNAASbusinessTrendAggrScriptName` (typo in script – both used) | Same as above; verify actual name in properties. |
| `MailId`, `ccList`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / headers for failure alerts. |
| `mnaas_retention_period_activations_aggr_daily`, `mnaas_retention_period_business_trend_daily_aggr` | Number of days/partitions to retain. |
| `MNAASDailyActivationsAggregationScriptName`, `MNAASDailyActivationsAggrScriptName` | Script name strings used in logs and alerts. |

If any of these variables are missing or empty, the script will fail early; a pre‑flight validation step is recommended.

---

### 8. Suggested Improvements (TODO)  

1. **Add a pre‑flight validation block** – before any processing, iterate over the required variables from the properties file and abort with a clear error if any are undefined or point to non‑existent files.  
2. **Replace ad‑hoc PID checks with `flock`** – wrap the entire script body in `flock -n /var/run/MNAAS_DailyActivations.lock` to guarantee single‑instance execution even if the status file becomes stale.  

--- 

*End of documentation.*