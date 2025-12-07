**File:** `move-mediation-scripts/bin/MNAAS_Daily_Tolling_tbl_Aggr.sh`  

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily aggregation of the *tolling* data set for the MNAAS (Move‑Mediation) platform. It runs three Java‑based steps in sequence (insert into the aggregation table, raw‑data retention, and aggregated‑data retention), refreshes the corresponding Impala/Hive tables, updates a shared process‑status file, writes detailed logs, and on failure raises an SDP ticket and sends an email notification. The script is intended to be executed by a cron job and is guarded against concurrent runs.

---

## 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **`MNAAS_insert_into_aggr_table`** | - Marks status flag = 1 in the process‑status file.<br>- Ensures the Java insert‑jar (`Insert_Daily_Aggr_table`) is not already running.<br>- Executes the jar with required arguments.<br>- On success, runs an Impala `REFRESH` for the aggregation table and logs success; on failure, logs and aborts. |
| **`MNAAS_retention_for_tolling_raw_daily`** | - Sets status flag = 2.<br>- Calls the generic `DropNthOlderPartition` Java class to drop old partitions from the raw tolling Hive table based on `mnaas_retention_period_tolling_raw_daily`.<br>- Refreshes the raw table in Impala and logs outcome. |
| **`MNAAS_retention_for_tolling_aggr_daily`** | - Sets status flag = 3.<br>- Similar to the raw‑retention step but operates on the aggregated tolling table using `mnaas_retention_period_tolling_aggr_daily`. |
| **`terminateCron_successful_completion`** | - Resets status flag = 0, marks job as *Success*, updates run‑time stamp, writes final log entries and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | - Logs failure, invokes `email_and_SDP_ticket_triggering_step`, exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | - Sets job status = *Failure* in the status file.<br>- Sends a formatted email to the MOVE dev team and creates an SDP ticket (via `mailx`) **once** per failure (guarded by `MNAAS_email_sdp_created` flag). |
| **Main block** | - Checks the PID stored in the status file to avoid concurrent runs.<br>- Reads the current `MNAAS_Daily_ProcessStatusFlag` and decides which subset of the three steps to execute (allows restart from a partially‑completed run).<br>- Updates the PID in the status file, logs start/end timestamps, and drives the workflow. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Daily_Tolling_tbl_Aggr.properties` – defines all environment variables used (paths, class names, DB hosts, retention periods, email addresses, etc.). |
| **Process‑status file** | `$MNAAS_Daily_Tolling_Aggr_ProcessStatusFileName` – a plain‑text key/value file that stores flags, PID, job status, timestamps, and email‑ticket flag. Updated throughout execution. |
| **Log file** | `$MNAAS_DailyTollingAggrLogPath` – appended with `set -x` output and explicit `logger` messages. |
| **Java JARs** | - `Insert_Daily_Aggr_table` (insert step)<br>- `DropNthOlderPartition` (retention steps) – both located via `$CLASSPATHVAR`, `$MNAAS_Main_JarPath`, `$Generic_Jar_Names`. |
| **Impala/Hive** | - `impala-shell` used to `REFRESH` tables (`${aggr_tolling_daily_tblname_refresh}`, `${tolling_daily_tblname_refresh}`).<br>- Assumes reachable Impala daemon (`$IMPALAD_HOST`) and Hive metastore (`$HIVE_HOST`, `$HIVE_JDBC_PORT`). |
| **Email/SDP** | - `mail` and `mailx` commands send failure notifications and create an SDP ticket. Requires proper MTA configuration and valid recipient addresses (`$MailId`, `$MOVE_DEV_TEAM`, etc.). |
| **External services** | - Impala daemon, Hive metastore, Java runtime, SMTP server. |
| **Assumptions** | - All required environment variables are defined in the properties file.<br>- Java classes are compatible with the current Hadoop/Impala version.<br>- The process‑status file is writable by the script user.<br>- No other process writes to the same status file concurrently. |
| **Outputs** | - Updated status file (flags, PID, timestamps).<br>- Log file with detailed execution trace.<br>- Refreshed Impala tables (visible to downstream analytics).<br>- Optional email and SDP ticket on failure. |

---

## 4. Integration Points & Connectivity  

| Component | Interaction |
|-----------|-------------|
| **Cron scheduler** | The script is invoked as a daily job (e.g., `0 2 * * * /path/MNAAS_Daily_Tolling_tbl_Aggr.sh`). |
| **Other MNAAS daily scripts** | Shares the same *process‑status* directory pattern (`*_ProcessStatusFileName`). The flag logic allows this script to be re‑run after a failure without re‑executing already‑completed steps, mirroring the pattern used in `MNAAS_Daily_SimInventory_tbl_Aggr.sh` and other daily aggregation scripts. |
| **Java processing layer** | Calls generic Java utilities (`Insert_Daily_Aggr_table`, `DropNthOlderPartition`) that are also used by other daily aggregation scripts (e.g., KYC feed, market management). |
| **Impala/Hive** | Refreshes tables that are consumed by downstream reporting pipelines (e.g., daily tolling reports, billing). |
| **Monitoring/Alerting** | Failure path triggers an SDP ticket (`insdp@tatacommunications.com`) and an email to the MOVE dev team – integrates with the organization’s incident‑management system. |
| **Process‑status file** | Acts as a lightweight coordination mechanism across the suite of daily MNAAS scripts; other scripts may read the same file to determine overall pipeline health. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID check may be bypassed if the status file is corrupted or the previous process hung. | Duplicate data loads, table locks, resource contention. | Add a timeout guard: if PID exists > X hours, treat as stale and clean up. |
| **Java job failure not captured** – exit code of the Java process is checked, but stdout/stderr are not logged. | Silent data loss or partial load. | Redirect Java stdout/stderr to the same log file (`>> $MNAAS_DailyTollingAggrLogPath 2>&1`). |
| **Impala refresh failure** – script proceeds to next step even if `impala-shell` returns non‑zero. | Stale data served to downstream consumers. | Capture the exit status of `impala-shell` and abort on failure. |
| **Email/SDP ticket spam** – repeated failures could generate many tickets if the flag is not reset. | Alert fatigue. | Ensure the flag (`MNAAS_email_sdp_created`) is cleared at the start of a new successful run (already done in `terminateCron_successful_completion`). |
| **Hard‑coded class names / table names** – any schema change requires property file update. | Script breakage. | Externalize class and table names into a central configuration service (e.g., ZooKeeper or a DB). |
| **Missing environment variables** – script will exit with cryptic errors. | Unplanned downtime. | Add a pre‑flight validation block that checks all required variables and exits with a clear message if any are missing. |

---

## 6. Running & Debugging the Script  

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Tolling_tbl_Aggr.properties` is present and readable.  
   - Verify Java, Impala client, and mail utilities are in the PATH.  
   - Confirm write permission on the log file path and the process‑status file.  

2. **Manual Execution**  
   ```bash
   export MNAASDailyTollingAggrScriptName=MNAAS_Daily_Tolling_tbl_Aggr.sh   # optional override
   ./MNAAS_Daily_Tolling_tbl_Aggr.sh
   ```
   - The script prints each command (`set -x`) and appends to the log file.  

3. **Debugging Tips**  
   - **Check the status file**: `cat $MNAAS_Daily_Tolling_Aggr_ProcessStatusFileName` to see current flag, PID, and job status.  
   - **Force a specific step**: Manually set `MNAAS_Daily_ProcessStatusFlag` to `1`, `2`, or `3` and re‑run to test a single function.  
   - **Inspect Java logs**: If the Java jar writes its own logs, locate them (often under `$MNAAS_DailyTollingAggrLogPath` or a jar‑specific path).  
   - **Validate Impala refresh**: Run the refresh query directly: `impala-shell -i $IMPALAD_HOST -q "REFRESH <table>"`.  
   - **Tail the log** while the script runs: `tail -f $MNAAS_DailyTollingAggrLogPath`.  

4. **Cron Integration**  
   - Add to crontab (example runs at 02:30 AM):  
     ```
     30 2 * * * /app/hadoop_users/MNAAS/bin/MNAAS_Daily_Tolling_tbl_Aggr.sh >> /dev/null 2>&1
     ```  
   - Ensure the cron user has the same environment (source the properties file or export needed vars in the crontab).  

---

## 7. External Config / Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_DailyTollingAggrLogPath` | Full path to the script’s log file. |
| `MNAAS_Daily_Tolling_Aggr_ProcessStatusFileName` | Path to the shared status file. |
| `Dname_MNAAS_Insert_Daily_Aggr_Tolling_tbl` | Process name used for PID check. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `Generic_Jar_Names` | Java classpath components. |
| `Insert_Daily_Aggr_table` | Main Java class for aggregation insert. |
| `MNAAS_Property_filename` | Property file passed to Java jobs (often the same as the Bash properties). |
| `MNAAS_Daily_Tolling_AggrCntrlFileName` | Control file consumed by the Java insert job. |
| `processname_tolling` | Identifier passed to Java job (used for logging inside the jar). |
| `aggr_tolling_daily_tblname_refresh` | Impala `REFRESH` statement for the aggregation table. |
| `IMPALAD_HOST` | Hostname of the Impala daemon. |
| `dbname`, `tolling_daily_tblname`, `tolling_aggr_daily_tblname` | Hive database and table names for retention steps. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive metastore connection details. |
| `mnaas_retention_period_tolling_raw_daily`, `mnaas_retention_period_tolling_aggr_daily` | Number of days/partitions to retain. |
| `tolling_daily_tblname_refresh`, `aggr_tolling_daily_tblname_refresh` | Impala `REFRESH` statements for the raw and aggregated tables. |
| `MailId`, `ccList`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email addresses used for failure notifications. |
| `MNAASDailyTollingAggrScriptName` | Human‑readable script identifier used in logs and tickets. |

If any of these variables are missing, the script will fail early (e.g., `sed` will error). Adding a validation block is recommended.

---

## 8. Suggested TODO / Improvements  

1. **Add Pre‑flight Validation** – before any processing, loop through a required‑variables list and abort with a clear error if any are undefined.  
2. **Capture Impala Refresh Exit Codes** – wrap each `impala-shell` call in a function that checks `$?` and triggers `terminateCron_Unsuccessful_completion` on non‑zero status, preventing downstream steps from running on stale data.  

---