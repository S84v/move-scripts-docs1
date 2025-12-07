**File:** `move-mediation-scripts/bin/MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh`  

---

## 1. One‑paragraph production summary
This Bash script orchestrates the daily load and aggregation of MSISDN‑level usage data for the MNAAS platform. It first inserts the latest raw usage records into a Hive/Impala aggregation table via a Java job, then runs a retention job that drops partitions older than a configurable retention period. Throughout the run it updates a shared *process‑status* file, logs progress, and on failure sends an email and creates an SDP ticket. The script is intended to be invoked by a daily cron schedule and protects against concurrent executions.

---

## 2. Key functions & responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_aggr_table** | * Updates the process‑status flag to *running* (value 1). <br>* Starts `msisdn_refresh.sh` in background. <br>* If the Java aggregation job (`$Insert_Daily_Aggr_table`) is not already running, launches it with the appropriate control files and classpath. <br>* On success runs an Impala `REFRESH` on the target table and logs success; on failure logs, timestamps, and aborts. |
| **MNAAS_retention_for_msisdn_level_daily_usage_aggr_daily** | * Sets the process‑status flag to *retention* (value 2). <br>* Executes the generic Java “DropNthOlderPartition” job to delete partitions older than `$mnaas_retention_period_MSISDN_level_daily_usage_aggr`. <br>* Refreshes Impala metadata; if the first refresh fails, runs an `INVALIDATE METADATA` followed by a second `REFRESH`. <br>* Logs success or aborts on error. |
| **terminateCron_successful_completion** | * Resets the status flag to *idle* (value 0) and marks the job as `Success`. <br>* Writes the run timestamp and clears the “email‑SDP‑created” flag. <br>* Emits final log entries and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | * Updates the run timestamp, logs failure, triggers email/SDP handling, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | * Marks the job status as `Failure`. <br>* Sends a pre‑formatted email to the configured distribution list (only once per failure) and logs ticket creation. |

---

## 3. Inputs, outputs & side‑effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties` – defines all environment variables used (log path, process‑status file, DB/Impala hosts, Java class names, jar locations, retention period, email recipients, etc.). |
| **Process‑status file** | `$MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` – a plain‑text key/value file that stores flags, PID, job status, timestamps, and email‑ticket flag. Updated throughout the run. |
| **Control files** | `$MNAAS_MSISDN_level_daily_usage_aggrCntrlFileName` – passed to the Java aggregation job. |
| **External services** | - **Hive / Impala** (tables referenced by `$msisdn_level_daily_usage_aggr_tblname`). <br>- **Java runtime** (jobs `Insert_Daily_Aggr_table` and `DropNthOlderPartition`). <br>- **Mail** (`mail` command) for failure notification. <br>- **SDP ticketing** (triggered indirectly via email). |
| **Side‑effects** | - Inserts rows into the aggregation Hive table. <br>- Drops old partitions (data deletion). <br>- Refreshes Impala metadata. <br>- Writes to the shared process‑status file. <br>- Generates log entries (`$MNAAS_MSISDN_level_daily_usage_aggrLogPath`). <br>- Sends email on failure. |
| **Assumptions** | - All variables referenced in the properties file are defined and point to accessible resources. <br>- Java classpath and JAR files are present and compatible with the Hadoop/Impala versions. <br>- The `ps` greps correctly identify running Java processes (no name collisions). <br>- The mail subsystem is functional and the `ccList`/`MailId` variables are set. |

---

## 4. Interaction with other scripts / components  

| Connected component | How it is used |
|---------------------|----------------|
| `msisdn_refresh.sh` (located under `MNAAS_CronFiles/`) | Launched in background by `MNAAS_insert_into_aggr_table` to refresh downstream caches or materialized views. |
| **Java JARs** (`$MNAAS_Main_JarPath`, `$Generic_Jar_Names`) | Contain the classes `Insert_Daily_Aggr_table` and `DropNthOlderPartition` that perform the heavy data‑movement and partition‑dropping logic. |
| **Hive/Impala tables** (`${dbname}.${msisdn_level_daily_usage_aggr_tblname}`) | Target of the aggregation insert and retention cleanup. |
| **Process‑status file** (shared with other MNAAS daily scripts) | Used by other scripts (e.g., `MNAAS_GBS_Load.sh`, `MNAAS_KYC_RNR_hourly_report.sh`) to coordinate execution windows and avoid overlapping runs. |
| **Cron scheduler** | The script is expected to be invoked by a daily cron entry (e.g., `0 2 * * * /app/hadoop_users/.../MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh`). |
| **Email/SDP** | Failure notification integrates with the broader incident‑management workflow. |

---

## 5. Operational risks & mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (multiple instances) | Duplicate inserts, race conditions on status file, partition loss. | The script checks the PID stored in the status file; ensure the status file is on a reliable shared filesystem and that the PID check is robust (e.g., use `pgrep -f` instead of simple `ps|grep`). |
| **Stale or corrupted status file** | Wrong flag values → skipped steps or endless loops. | Add validation at start (verify required keys exist, PID is numeric). Consider rotating the status file daily. |
| **Java job failure** (OOM, classpath mismatch) | No data loaded, downstream reports missing. | Capture Java exit code (already done) and forward the stack trace to the log. Alert on repeated failures. |
| **Impala refresh failure** | New partitions not visible to downstream queries. | The script already attempts `INVALIDATE METADATA` on first failure; monitor the second refresh outcome. |
| **Mail delivery failure** | Operators not notified of failures. | Verify mail server health; fallback to writing a flag file that a monitoring daemon can poll. |
| **Retention over‑deletion** (wrong retention period) | Permanent loss of historical usage data. | Store the retention period in a centrally‑managed config and enforce a minimum value via a sanity check before invoking the drop job. |
| **Missing environment variables** | Script aborts early with cryptic errors. | Add a pre‑flight check that prints missing variables and exits with a clear message. |

---

## 6. Running / debugging the script  

**Typical execution (via cron):**  

```bash
# As the Hadoop user that owns the MNAAS environment
sh /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh
```

**Manual run (for testing):**  

1. Export any missing variables that are normally sourced from the properties file, or point `MNAAS_Property_Files/MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties` to a test version.  
2. Run with `bash -x` (the script already contains `set -x`) to see each command.  
3. Tail the log file defined by `$MNAAS_MSISDN_level_daily_usage_aggrLogPath` while the script runs.  

**Debugging tips:**  

| Step | Action |
|------|--------|
| **Check PID handling** | Verify the PID stored in the status file matches the actual Java process (`ps -p <pid>`). |
| **Inspect Java logs** | The Java jobs usually write to `$MNAAS_MSISDN_level_daily_usage_aggrLogPath`; look for stack traces. |
| **Validate Impala commands** | Run the generated refresh statements manually in `impala-shell` to ensure they succeed. |
| **Force failure** | Temporarily set an invalid classpath or retention period to test the failure path and email generation. |
| **Status file sanity** | `cat $MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` should show keys: `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_Daily_ProcessName`, `MNAAS_Script_Process_Id`, `MNAAS_job_status`, `MNAAS_job_ran_time`, `MNAAS_email_sdp_created`. |

---

## 7. External configuration & environment variables  

| Variable (populated in the `.properties` file) | Purpose |
|-----------------------------------------------|---------|
| `MNAAS_MSISDN_level_daily_usage_aggrLogPath` | Full path to the script’s log file. |
| `MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` | Path to the shared status file. |
| `MNAAS_MSISDN_Level_Daily_Usage_Aggr_ScriptName` | Human‑readable script identifier used in logs/emails. |
| `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl` | Name used for the Java process (`-Dname=`). |
| `Insert_Daily_Aggr_table` | Fully‑qualified Java class that performs the aggregation insert. |
| `DropNthOlderPartition` | Fully‑qualified Java class that drops old partitions. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `Generic_Jar_Names` | Classpath and JAR locations for the Java jobs. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java). |
| `MNAAS_MSISDN_level_daily_usage_aggrCntrlFileName` | Control file for the aggregation job. |
| `processname_msisdn_level_daily_usage_aggr` | Identifier passed to Java (likely for logging). |
| `MNAAS_MSISDN_Aggr_Partitions_Daily_Uniq_FileName` | File containing partition list (used by Java). |
| `dbname`, `msisdn_level_daily_usage_aggr_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive connection details (used by Java). |
| `IMPALAD_HOST` | Impala daemon host for `impala-shell` commands. |
| `mnaas_retention_period_MSISDN_level_daily_usage_aggr` | Number of days to retain data (passed to retention job). |
| `MailId`, `ccList` | Email recipients for failure notifications. |
| `MNAAS_email_sdp_created` | Flag to avoid sending duplicate failure emails/tickets. |

---

## 8. Suggested improvements (TODO)

1. **Robust PID detection** – replace the current `ps | grep` pattern with `pgrep -f "^java.*$Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl"` to avoid false positives from the `grep` process itself.
2. **Pre‑flight validation** – add a function that checks the existence and readability of all required variables, files, JARs, and external services before any processing begins; abort early with a clear error message if any check fails.  

---