**File:** `move-mediation-scripts\bin\MNAAS_MSISDN_Level_Daily_Usage_Aggr_apr21_mismatch.sh`  

---

## 1. Purpose (one‑paragraph summary)

This script orchestrates the daily load and aggregation of MSISDN‑level usage data for the MNAAS (Mobile Network Analytics & Assurance System) platform. It first inserts fresh usage records into a Hive/Impala table via a Java job, then runs a retention job that drops partitions older than the configured retention period. The script maintains a lightweight process‑status file to guarantee single‑instance execution, logs all activity, and on failure raises an SDP ticket and sends an email alert. It is intended to be run once per day (typically via cron) after the “msisdn_refresh” refresh job has completed.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_aggr_table** | * Sets process‑status flag to *1* (insert phase). <br>* Starts `msisdn_refresh.sh` in background. <br>* If the Java aggregation job (`Insert_Daily_Aggr_table`) is not already running, launches it with the appropriate classpath, property file, control file, and partition‑definition file. <br>* On success, issues an `impala-shell` refresh of the target table and logs success; on failure, logs error and aborts. |
| **MNAAS_retention_for_msisdn_level_daily_usage_aggr_daily** | * Sets process‑status flag to *2* (retention phase). <br>* If the retention Java job (`DropNthOlderPartition`) is not already running, executes it to drop partitions older than the configured retention period. <br>* Refreshes Impala metadata; if the first refresh fails, runs an invalidate‑metadata followed by a refresh. <br>* Logs success or failure and aborts on error. |
| **terminateCron_successful_completion** | * Resets process‑status flag to *0* (idle) and updates job‑status to *Success* with a timestamp. <br>* Writes a final “script terminated successfully” entry to the log and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | * Updates the run‑time stamp, logs failure, triggers email/SDP ticket, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | * Marks job‑status as *Failure* in the status file. <br>* Sends a pre‑formatted email to the configured distribution list (only once per failure) and records that an SDP ticket has been created. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties` – defines all environment variables used throughout the script (paths, jar names, class names, DB hosts, table names, retention period, email lists, etc.). |
| **Process‑status file** | `$MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` – a simple key/value file used to store PID, flags, job status, timestamps, and email‑sent flag. |
| **Log file** | `$MNAAS_MSISDN_level_daily_usage_aggrLogPath` – appended with all `logger` output and script‑level messages. |
| **External binaries / services** | - Java runtime (runs two JARs: `$Insert_Daily_Aggr_table` and `$DropNthOlderPartition`). <br>- Hive/Impala (`impala-shell` for metadata refresh). <br>- `msisdn_refresh.sh` (background refresh job). <br>- Linux `mail` command for alerts. |
| **Outputs** | - Populated/updated Hive/Impala table `${msisdn_level_daily_usage_aggr_tblname}`. <br>- Retained partitions only up to `$mnaas_retention_period_MSISDN_level_daily_usage_aggr`. <br>- Updated status file (flags, timestamps, job status). |
| **Assumptions** | - All variables referenced in the properties file are defined and point to accessible resources. <br>- The Java JARs are compatible with the current Hadoop/Impala versions. <br>- The process‑status file is writable by the script user. <br>- `ps -ef | grep $Dname_*` reliably detects running Java processes (no false positives). <br>- Email relay is functional and `mail` is installed. |

---

## 4. Interaction with Other Components

| Component | How this script connects |
|-----------|--------------------------|
| **`msisdn_refresh.sh`** | Launched in background at the start of the insert phase to refresh raw MSISDN data before aggregation. |
| **Java aggregation JAR (`Insert_Daily_Aggr_table`)** | Performs the heavy‑weight data load from staging to the daily aggregation Hive table. |
| **Java retention JAR (`DropNthOlderPartition`)** | Executes partition‑dropping based on the retention period. |
| **Impala (`impala-shell`)** | Used twice: after insert to refresh table metadata, and after retention (with optional invalidate‑metadata) to make new partitions visible. |
| **Process‑status file** | Shared with other MNAAS daily scripts (e.g., `MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh`) to coordinate run state and avoid concurrent executions. |
| **Email/SDP system** | Sends alerts and creates tickets on failure; downstream ticketing system consumes the email content. |
| **Cron scheduler** | Typically invoked by a daily cron entry (e.g., `0 2 * * * /path/.../MNAAS_MSISDN_Level_Daily_Usage_Aggr_apr21_mismatch.sh`). |

---

## 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Concurrent runs** – two instances could start if the status file is stale or PID check fails. | Ensure the status file is atomically updated; add a lockfile (`flock`) around the whole script. |
| **Java job hangs or crashes** – script may wait indefinitely or mis‑detect running processes. | Implement a timeout wrapper (`timeout` command) around Java calls; monitor Java process exit codes more robustly. |
| **Impala metadata refresh failure** – could leave stale partitions visible. | After a failed refresh, automatically run `invalidate metadata` followed by a second refresh (already partially implemented). |
| **Missing or corrupted properties file** – leads to undefined variables. | Add a sanity‑check after sourcing the properties file that verifies required variables are non‑empty; abort early with a clear error. |
| **Email spam / duplicate tickets** – repeated failures may flood inboxes. | The status file flag `MNAAS_email_sdp_created` already prevents duplicate alerts; ensure it is reset on next successful run. |
| **Log file growth** – unbounded log size over time. | Rotate logs via `logrotate` or truncate the file at the start of each run. |
| **Permission issues** – script may lack rights to write to HDFS, status file, or send mail. | Run the script under a dedicated service account with the minimal required privileges; verify permissions during deployment. |

---

## 6. Running / Debugging the Script

| Step | Command / Action |
|------|------------------|
| **Standard execution (via cron)** | Cron entry typically: <br>`0 2 * * * /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_MSISDN_Level_Daily_Usage_Aggr_apr21_mismatch.sh >> /dev/null 2>&1` |
| **Manual run (debug mode)** | `bash -x /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_MSISDN_Level_Daily_Usage_Aggr_apr21_mismatch.sh` <br>`-x` prints each command and variable expansion to stdout. |
| **Check current status** | `cat $MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` |
| **Force a clean start** | Delete or reset the status file (ensure no other script is using it) and remove any stale Java processes: <br>`pkill -f $Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl` |
| **Inspect logs** | `tail -f $MNAAS_MSISDN_level_daily_usage_aggrLogPath` |
| **Validate Java job execution** | Verify the JAR classpath and arguments: <br>`java -cp $CLASSPATHVAR:$MNAAS_Main_JarPath $Insert_Daily_Aggr_table ...` – run manually with `--help` if supported. |
| **Test email alert** | Temporarily set `MNAAS_email_sdp_created=No` in the status file and force a failure (e.g., `exit 1` after a step) to confirm mail is sent. |

---

## 7. External Configuration & Environment Variables

| Variable (populated in the `.properties` file) | Meaning / Usage |
|-----------------------------------------------|-----------------|
| `MNAAS_MSISDN_level_daily_usage_aggrLogPath` | Full path to the script’s log file. |
| `MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` | Path to the shared status/lock file. |
| `MNAAS_MSISDN_Level_Daily_Usage_Aggr_ScriptName` | Human‑readable script identifier used in logs/emails. |
| `MNAAS_Main_JarPath`, `Generic_Jar_Names` | Locations of the Java JARs containing the aggregation and retention classes. |
| `Insert_Daily_Aggr_table`, `DropNthOlderPartition` | Fully qualified Java class names to invoke. |
| `CLASSPATHVAR` | Additional classpath entries required by the Java jobs. |
| `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl`, `Dname_MNAAS_retention_msisdn_level_daily_usage_aggr_tbl` | Unique process names used for `ps` detection. |
| `msisdn_level_daily_usage_aggr_tblname`, `msisdn_level_daily_usage_aggr_tblname_refresh`, `msisdn_level_daily_usage_aggr_tblname_invalidate_metadata` | Hive/Impala table name and associated refresh/metadata commands. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Hosts/ports for Impala and Hive services. |
| `mnaas_retention_period_MSISDN_level_daily_usage_aggr` | Number of days to retain partitions. |
| `MailId`, `ccList` | Email recipients for failure notifications. |
| `MNAAS_Property_filename`, `MNAAS_MSISDN_level_daily_usage_aggrCntrlFileName` | Additional property/control files passed to Java jobs. |
| `processname_msisdn_level_daily_usage_aggr` | Identifier passed to the aggregation Java class. |
| `MNAAS_MSISDN_Aggr_Partitions_Daily_Uniq_Apr_mismatch` | File that defines the partitioning scheme for the April‑2021 mismatch fix. |

---

## 8. Suggested Improvements (TODO)

1. **Add a robust lock mechanism** – replace the ad‑hoc status‑file PID check with `flock` (or a Zookeeper lock) to guarantee single‑instance execution even across node failures.  
2. **Introduce timeout handling for Java jobs** – wrap each `java` invocation with the GNU `timeout` utility (e.g., `timeout 2h java …`) and handle the timeout exit code to avoid indefinite hangs.

---