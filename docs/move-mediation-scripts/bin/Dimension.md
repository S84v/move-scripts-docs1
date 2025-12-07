**File:** `move-mediation-scripts/bin/Dimension.sh`

---

## 1. High‑Level Summary
`Dimension.sh` is a production‑grade Bash driver that populates two time‑related dimension tables in Hive/Impala (a *time* dimension and a *date‑interval* dimension) for the MNAAS (Move‑NAA‑S) data‑move platform. It reads a property file for configuration, invokes Java classes that generate the table data, writes status flags to a shared process‑status file, logs progress, and on failure sends an email and creates an SDP ticket. The script is intended to be run daily (or monthly) by a scheduler (e.g., cron) and includes safeguards to prevent concurrent executions.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `MNAAS_insert_into_dimension_table` | - Sets process flag = 1 (dimension load) in the status file.<br>- Computes current date (`%Y-%m-%d`).<br>- Clears the temporary output file (`$time_dimension_file`).<br>- Executes the Java class `$Dimension_class_name` to generate the *time* dimension data and write it to `$time_dimension_file`.<br>- Logs success/failure; on failure calls `terminateCron_Unsuccessful_completion`. |
| `MNAAS_insert_into_dateinterval_table` | - Sets process flag = 2 (date‑interval load) in the status file.<br>- Computes current date.<br>- Clears `$date_interval_file`.<br>- Executes the Java class `$Dateinterval_class_name` to generate the *date‑interval* dimension data.<br>- Logs success/failure; on failure calls `terminateCron_Unsuccessful_completion`. |
| `terminateCron_successful_completion` | - Resets status flags to “idle” (`MNAAS_Daily_ProcessStatusFlag=0`).<br>- Marks job status = Success, clears email‑sent flag, records run time.<br>- Writes final log entries and exits with status 0. |
| `terminateCron_Unsuccessful_completion` | - Logs failure, invokes `email_and_SDP_ticket_triggering_step`, exits with status 1. |
| `email_and_SDP_ticket_triggering_step` | - Sets job status = Failure.<br>- If an SDP ticket/email has not yet been sent (`MNAAS_email_sdp_created=No`), sends a notification email to the MOVE dev team and creates an SDP ticket via `mailx`.<br>- Updates the status file to record ticket‑creation time and flag. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/Dimension.properties` – defines all `$MNAAS_*`, `$CLASSPATHVAR`, `$Generic_Jar_Names`, `$Dimension_class_name`, `$Dateinterval_class_name`, Hive/Impala hosts, ports, DB/table names, email/ticket variables, etc. |
| **Process‑status file** | `$MNAAS_Dimension_ProcessStatusFileName` – a plain‑text key/value file used for inter‑script coordination, PID tracking, flag values, job status, email‑sent flag, run‑time stamp. Modified throughout the script. |
| **Log file** | `$MNAAS_DimensionLogPath` – appended with all `logger` output and error messages (`exec 2>>$MNAAS_DimensionLogPath`). |
| **Temporary data files** | `$time_dimension_file` and `$date_interval_file` – cleared before each Java run; populated by the Java classes; later consumed by Hive/Impala (outside this script). |
| **External services** | - **Java runtime** (`java -cp …`) – runs two custom JAR classes that write the dimension data.<br>- **Hive/Impala** – target tables (`$dbname.$time_dimension_tblname`, `$dbname.$date_interval_tblname`). Refresh commands are present but commented out.<br>- **Mail** (`mail`, `mailx`) – sends failure notifications and creates SDP tickets.<br>- **Operating system** – `ps` for PID check, `sed` for file edits, `logger` for syslog. |
| **Assumptions** | - All environment variables are defined in the sourced properties file.<br>- The Java classes are present in `$Generic_Jar_Names` and are compatible with the supplied arguments.<br>- The process‑status file exists and is writable by the script user.<br>- Network connectivity to Hive/Impala hosts and to the mail server is available.<br>- The script runs under a user with permission to write logs, edit status file, and execute Java. |

---

## 4. Interaction with Other Components

| Component | Interaction Point |
|-----------|-------------------|
| **Other MNAAS scripts** | All MNAAS jobs share the same `$MNAAS_Dimension_ProcessStatusFileName`. Flags set here (`MNAAS_Daily_ProcessStatusFlag`) are read by downstream or upstream scripts to decide whether to run. |
| **Hive/Impala tables** | The Java classes load data into `$dbname.$time_dimension_tblname` and `$dbname.$date_interval_tblname`. Other analytics pipelines (e.g., Tableau, MSPS) later query these tables. |
| **Ticketing / Notification system** | Failure handling uses the SDP ticketing email address (`$SDP_ticket_from_email`) and distribution list (`$SDP_Receipient_List`). The same email format is used by other Move scripts for consistency. |
| **Scheduler (cron)** | The script is intended to be invoked by a cron entry; the PID guard prevents overlapping runs, a pattern used across the Move codebase. |
| **Global utilities** | The script relies on common utility scripts (`globalDef.js`, `mongo.js`, etc.) for shared constants, though not directly imported here; they may be referenced indirectly via the properties file. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale PID in status file** – if the script crashes without clearing `MNAAS_Script_Process_Id`, subsequent runs will be blocked. | Production outage (dimension tables not refreshed). | Add a watchdog that validates the PID is still alive (`kill -0 $PID`) before assuming the process is running; clear the PID on abnormal exit. |
| **Missing or malformed property values** – undefined `$CLASSPATHVAR`, `$Dimension_class_name`, etc. | Immediate failure, no logs beyond generic error. | Validate required variables at script start; abort with clear error if any are empty. |
| **Java class failure** – due to schema change or Hive connectivity. | Dimension tables incomplete; downstream analytics break. | Capture Java exit code (already done) and log the full stdout/stderr to a separate file for post‑mortem. Consider retry logic for transient Hive errors. |
| **Log file growth** – continuous `exec 2>>` appends without rotation. | Disk exhaustion. | Configure logrotate for `$MNAAS_DimensionLogPath` or switch to syslog with size limits. |
| **Email/ticket flood** – repeated failures could generate many tickets. | Alert fatigue. | Ensure `MNAAS_email_sdp_created` flag is reliably persisted; add a throttling counter (e.g., max 3 tickets per day). |
| **Permission issues on status or data files** – especially after OS/user changes. | Script cannot update flags or write data. | Run a pre‑flight permission check; document required file ownership (`chmod 664` etc.). |

---

## 6. Running / Debugging the Script

1. **Manual invocation**  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./Dimension.sh
   ```
   Ensure the user has read access to `Dimension.properties` and write access to the log and status files.

2. **Check environment**  
   ```bash
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/Dimension.properties
   env | grep MNAAS
   ```
   Verify that all `$MNAAS_*` variables are set.

3. **Tail the log** while the script runs:  
   ```bash
   tail -f $MNAAS_DimensionLogPath
   ```

4. **Inspect the status file** before and after execution to confirm flag transitions:  
   ```bash
   cat $MNAAS_Dimension_ProcessStatusFileName
   ```

5. **Debug Java step** – if the Java class fails, run it directly with the same arguments to capture stdout/stderr:  
   ```bash
   java -cp $CLASSPATHVAR:$Generic_Jar_Names $Dimension_class_name 2023-09-01 mydb time_dim hive_host 10000 /tmp/time_dim.out /path/to/log
   ```

6. **Force a clean start** (e.g., after a crash):  
   ```bash
   sed -i 's/^MNAAS_Script_Process_Id=.*/MNAAS_Script_Process_Id=0/' $MNAAS_Dimension_ProcessStatusFileName
   ```

7. **Exit codes** – `0` = success, `1` = failure (used by the scheduler to trigger alerts).

---

## 7. External Config / Environment Variables

| Variable (from `Dimension.properties`) | Purpose |
|----------------------------------------|---------|
| `MNAAS_DimensionLogPath` | Path to the script’s log file (stderr redirected here). |
| `MNAAS_Dimension_ProcessStatusFileName` | Shared status/flag file used for coordination and PID tracking. |
| `time_dimension_file` / `date_interval_file` | Temporary files that receive Java output before Hive load. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath and JAR bundle containing the dimension generators. |
| `Dimension_class_name`, `Dateinterval_class_name` | Fully‑qualified Java class names invoked by the script. |
| `dbname`, `time_dimension_tblname`, `date_interval_tblname` | Hive database and table identifiers. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive connection details passed to Java. |
| `IMPALAD_HOST` (currently unused) | Intended Impala host for optional `refresh` commands. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `SDP_Receipient_List` | Email addresses / distribution lists for failure notifications and SDP ticket creation. |
| `MNAASDimensionScriptName` | Human‑readable script identifier used in logs and tickets. |

*If any of these variables are missing, the script will abort or behave unpredictably.*

---

## 8. Suggested Improvements (TODO)

1. **PID Validation & Cleanup** – Add a function that checks whether the PID stored in the status file is still alive (`kill -0 $PID`). If not, clear the PID and allow a new run. This prevents dead‑lock after an ungraceful termination.

2. **Centralised Configuration Validation** – Implement a pre‑run block that iterates over a required‑variables list, aborting with a clear error message if any are empty. This makes troubleshooting faster and avoids silent failures downstream.

*(Both changes can be added without altering the existing business logic and will improve reliability.)*