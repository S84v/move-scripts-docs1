**File:** `move-mediation-scripts/bin/MNAAS_Activations_tbl_load_driver.sh`

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily load and aggregation of MNAAS activation records into Hive/Impala tables. It reads a property file for configuration, updates a shared status file, invokes a per‑feed aggregation script for each activation source, runs Java‑based partition‑retention jobs, refreshes Impala metadata, and handles success/failure notification (email + SDP ticket). The script also guards against concurrent executions by checking a PID stored in the status file.

---

## 2. Key Functions / Logical Blocks  

| Function / Block | Responsibility |
|------------------|----------------|
| **`MNAAS_File_Processing`** | Loops over all activation feed sources defined in the associative array `FEED_FILE_PATTERN_ACTIVATIONS`; for each source logs start, calls the aggregation shell script (`$MNAASDailyActivationsAggregationScriptName`) with the feed pattern and previous processed file path, and aborts on any non‑zero exit code. |
| **`MNAAS_retention_for_activations_raw_daily`** | Updates status flag, runs a Java class (`$DropNthOlderPartition`) to drop N older partitions from the raw daily activations table, then refreshes the Impala table (`$activations_daily_tblname_refresh`). |
| **`MNAAS_retention_for_activations_file_record_count`** | Similar to the previous block but operates on the *file‑record‑count* table, using the same Java class and a different Impala refresh query (`$aggr_activations_daily_tblname_refresh`). |
| **`terminateCron_successful_completion`** | Writes a “Success” state to the status file, records run time, logs completion, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Sends a failure notification email to the MOVE dev team and creates an SDP ticket (via `mailx`) if a ticket has not already been generated for this run. Updates status file flags accordingly. |
| **Main script block** | Checks for an already‑running instance using the PID stored in the status file, writes its own PID, decides which steps to run based on the current `MNAAS_Combined_ProcessStatusFlag`, and finally calls the appropriate functions. |

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Activations_tbl_load_driver.properties` – defines all `$MNAAS_*` variables, associative arrays, classpaths, JAR names, Hive/Impala hosts, retention periods, email lists, etc. |
| **Environment variables** | None required beyond those set in the properties file; the script uses `$hour`, `$min` derived locally. |
| **External files** | • Status file: `$MNAAS_Daily_Activations_Load_CombinedActivationStatuFile` (plain‑text key/value). <br>• Log file: `$MNAAS_DailyActivationsCombinedLoadLogPath`. |
| **External scripts** | `$MNAASDailyActivationsAggregationScriptName` (called per feed). |
| **Java class** | `$DropNthOlderPartition` (executed via `java -cp …`). |
| **Impala** | `impala-shell` commands to refresh table metadata. |
| **Mail / SDP** | Uses `mail` and `mailx` to send failure alerts and create SDP tickets. |
| **Side‑effects** | • Updates status file flags and timestamps. <br>• Writes extensive log entries. <br>• May delete old Hive partitions (retention). <br>• Sends email / creates SDP ticket on failure. |
| **Assumptions** | • All paths and variables defined in the properties file exist and are accessible. <br>• The aggregation script and Java JAR are compatible with the current Hive/Impala versions. <br>• The host running the script has network access to Hive, Impala, and the mail/SDP services. <br>• Only one instance runs at a time (enforced by PID check). |

---

## 4. Interaction with Other Components  

| Component | How this script connects |
|-----------|--------------------------|
| **`MNAAS_Activations_tbl_load_driver.properties`** | Provides all runtime configuration (paths, DB names, hostnames, retention periods, email lists). |
| **Aggregation script (`$MNAASDailyActivationsAggregationScriptName`)** | Called for each feed source; responsible for ingesting raw activation files into a staging area / Hive table. |
| **Java JAR (`$Generic_Jar_Names`)** | Contains the `DropNthOlderPartition` class used for partition cleanup. |
| **Hive / Impala** | The script invokes `impala-shell` to refresh metadata after partition changes. |
| **Status file** | Shared with other MNAAS scripts (e.g., `MNAAS_Activations_tbl_Load.sh`) to coordinate multi‑step processing and avoid duplication. |
| **Mail / SDP ticketing** | Failure notifications are sent to the MOVE dev team and to the SDP ticketing system (`insdp@tatacommunications.com`). |
| **Cron / Scheduler** | Typically executed by a daily cron job; the script itself checks for concurrent runs. |
| **Logging infrastructure** | Writes to `$MNAAS_DailyActivationsCombinedLoadLogPath`, which is likely collected by a central log aggregator (e.g., Splunk, ELK). |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or malformed properties file** | Script aborts early, no processing. | Validate existence and syntax of the properties file at start; exit with clear error code if validation fails. |
| **Log or status file not writable** | No audit trail; status may become stale, leading to duplicate runs. | Ensure directory permissions; add a pre‑run check (`touch`/`chmod`) and abort if not possible. |
| **Concurrent executions** | Duplicate data loads, race conditions on status file. | PID check already present; reinforce by using a lock file (`flock`) as an extra safeguard. |
| **Java partition‑drop failure** | Old partitions remain, causing storage bloat; downstream queries may see stale data. | Capture Java stdout/stderr to a separate log; add retry logic or alert on non‑zero exit. |
| **Impala refresh failure** | New partitions not visible to queries. | Verify `impala-shell` return code; if failure, retry a limited number of times before aborting. |
| **Email/SDP ticket delivery failure** | Operators may miss critical failure alerts. | Log the result of `mail`/`mailx`; if they return non‑zero, write to a fallback alert file or trigger a monitoring alarm. |
| **Hard‑coded hour check (`if [ $hour = $hour_var ]`)** | If `$hour_var` is mis‑configured, retention jobs may never run. | Document required value; consider making the check configurable or removing it if unnecessary. |
| **Unbounded associative array iteration** | If a new feed is added without proper script, the driver may call a non‑existent aggregation script. | Validate that `$MNAASDailyActivationsAggregationScriptName` exists before invoking; log missing scripts as errors. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the properties file exists and is readable.  
   - Verify that `$MNAAS_Daily_Activations_Load_CombinedActivationStatuFile` and `$MNAAS_DailyActivationsCombinedLoadLogPath` are writable by the user running the script.  
   - Confirm that the aggregation script, Java JARs, Hive/Impala hosts, and mail utilities are reachable.

2. **Typical Invocation (via cron)**  
   ```bash
   # Example cron entry (run daily at 02:15)
   15 2 * * * /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_Activations_tbl_load_driver.sh >> /var/log/mnaas_driver_cron.log 2>&1
   ```

3. **Manual Run (for testing)**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts
   ./MNAAS_Activations_tbl_load_driver.sh
   ```

4. **Debugging Steps**  
   - **Check PID lock**: `cat $MNAAS_Daily_Activations_Load_CombinedActivationStatuFile | grep MNAAS_Script_Process_Id`.  
   - **Inspect status flag**: `grep MNAAS_Combined_ProcessStatusFlag $MNAAS_Daily_Activations_Load_CombinedActivationStatuFile`.  
   - **Tail the log** while the script runs: `tail -f $MNAAS_DailyActivationsCombinedLoadLogPath`.  
   - **Force a specific step** by manually setting the flag in the status file (e.g., `echo "MNAAS_Combined_ProcessStatusFlag=1" >> $MNAAS_Daily_Activations_Load_CombinedActivationStatuFile`).  
   - **Validate Java execution**: run the Java command directly with sample parameters to ensure classpath and JAR are correct.  
   - **Check Impala refresh**: execute the same `impala-shell -i $IMPALAD_HOST -q "<refresh_sql>"` manually and verify output.

5. **Post‑run verification**  
   - Confirm that new partitions appear in Hive (`SHOW PARTITIONS <table>`).  
   - Verify that the status file now shows `MNAAS_job_status=Success`.  
   - Ensure no new SDP ticket was created unless a failure occurred.

---

## 7. External Config / Environment References  

| Variable (populated in properties file) | Purpose |
|------------------------------------------|---------|
| `MNAAS_Daily_Activations_Load_CombinedActivationStatuFile` | Central status/flag file shared across MNAAS scripts. |
| `MNAAS_DailyActivationsCombinedLoadLogPath` | Path to the consolidated log file for this driver. |
| `FEED_FILE_PATTERN_ACTIVATIONS` (associative array) | Mapping of feed source names to filename glob patterns. |
| `MNAASDailyActivationsAggregationScriptName` | Name of the per‑feed aggregation script invoked in the loop. |
| `MNAASDailyActivationsProcessingScript` | Human‑readable script identifier used in log/email subjects. |
| `MNAASShellScriptPath` | Directory containing the aggregation script. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath and JAR list for the retention job. |
| `DropNthOlderPartition` | Fully‑qualified Java class that drops old Hive partitions. |
| `dbname`, `activations_daily_tblname`, `activations_file_record_count_tblname` | Hive database and table names used by retention jobs. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive connection details for the Java job. |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `activations_daily_tblname_refresh`, `aggr_activations_daily_tblname_refresh` | Impala `REFRESH` statements (or full DDL) executed after partition changes. |
| `mnaas_retention_period_activations_raw_daily`, `mnaas_retention_period_activations_file_record_count` | Number of days/partitions to retain. |
| `hour_var` | Hour of day when retention jobs are allowed to run (e.g., `"02"`). |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / headers for failure notifications. |
| `insdp@tatacommunications.com` | SDP ticketing mailbox. |

---

## 8. Suggested Improvements (TODO)

1. **Add Robust Parameter Validation** – At script start, verify that every required variable from the properties file is set and non‑empty; exit early with a clear error message if any are missing.

2. **Replace Manual PID/Flag File Logic with `flock`** – Use `flock -n /tmp/mnaas_driver.lock` to guarantee exclusive execution and simplify the concurrency guard, while still updating the status file for downstream scripts.

---