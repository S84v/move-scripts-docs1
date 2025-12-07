**File:** `move-mediation-scripts/bin/MNAAS_business_trend_Aggr.sh`  

---

## 1. Purpose (one‑paragraph summary)

This Bash script orchestrates the monthly “business trend” data pipeline for the MNAAS (Move‑N‑Analytics‑As‑a‑Service) platform. It creates a raw aggregation file via a Java job, removes stale Hive partitions, loads the raw data into a temporary non‑partitioned table on HDFS, copies it into an intermediate Hive/Impala table, and finally runs a second Java job that aggregates the data into the final daily business‑trend table. Throughout the run it updates a shared process‑status file, logs progress, and on failure generates an SDP ticket and email notification.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_creation_of_file** | Executes the Java class that extracts raw business‑trend data for the supplied `event_date` and writes it to a local file (`$MNAAS_business_Trend_Data_OP_filename`). Updates status flag = 1. |
| **MNAAS_remove_partitions_from_the_table** | Calls a generic Java utility to drop Hive partitions for the target daily aggregation table (`$business_trend_daily_aggr_tblname`) for the given date. Refreshes Impala metadata. Updates status flag = 2. |
| **MNAAS_business_inter_table_loading** | Clears the HDFS staging path, copies the raw file to HDFS, then runs a Java loader to insert the data into an intermediate non‑partitioned Hive table (`$business_trend_inter_daily_aggr_tblname`). Refreshes Impala metadata. Updates status flag = 3. |
| **MNAAS_business_aggr_table_loading** | Executes the final Java aggregation job that reads from the intermediate table and writes to the daily business‑trend aggregation table. Refreshes Impala metadata. Updates status flag = 4. |
| **terminateCron_successful_completion** | Resets the status file to “Success”, clears the running‑process flag, writes a completion timestamp, logs the success and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Updates the status file to “Failure”, checks whether an SDP ticket/email has already been sent, and if not sends a templated email and creates an SDP ticket via `mailx`. Marks ticket‑sent flag to avoid duplicates. |
| **Main script block** | Prevents concurrent runs by checking the PID stored in the status file, reads the current flag, and drives the functions in the correct order to resume from the last successful step. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input arguments** | `$1` – `event_date` (expected format: `YYYY-MM-DD`). |
| **Configuration file** | `MNAAS_business_trend_Aggr.properties` (sourced at start). Contains all environment‑specific variables (paths, class names, DB/host names, table names, email lists, etc.). |
| **Process‑status file** | `$MNAAS_business_trend_Aggr_ProcessStatusFileName` – a simple key‑value file used for coordination, PID tracking, and flag management. |
| **Log file** | `$MNAAS_business_trend_AggrLogPath` – appended with all `logger` output and Bash `set -x` trace. |
| **Generated files** | - Raw aggregation file: `$MNAAS_business_Trend_Data_OP_filename` (local).<br>- HDFS staging directory: `$MNAAS_Daily_Aggregation_businesstrend_PathName`. |
| **External services** | - **Hadoop HDFS** (file removal & copy).<br>- **Hive** (partition drop, table loads).<br>- **Impala** (metadata refresh via `impala-shell`).<br>- **Java runtime** (multiple JARs).<br>- **Mail / SDP ticketing** (via `mail` and `mailx`). |
| **Side effects** | - Alters Hive/Impala tables (partition removal, data inserts, metadata refresh).<br>- Sends email & creates SDP ticket on failure.<br>- Updates shared status file that may be read by other scripts or monitoring tools. |
| **Assumptions** | - All referenced JARs, class names, and Hadoop/Hive/Impala hosts are reachable and the correct versions are on the classpath.<br>- The properties file defines every variable used; missing entries will cause script failure.<br>- The user executing the script has sufficient OS, HDFS, Hive, and Impala permissions.<br>- Only one instance runs at a time (enforced by PID check). |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Process‑status file** | Shared with other MNAAS scripts (e.g., traffic aggregation scripts) that also read/write `MNAAS_*_ProcessStatusFileName`. It provides a global view of the pipeline state. |
| **Common Java utilities** | The `drop_partition_in_table` and `Load_nonpart_table` classes are generic utilities used by multiple aggregation scripts (e.g., `MNAAS_Traffic_*`). |
| **Impala/Hive tables** | The tables `business_trend_daily_aggr_tblname` and `business_trend_inter_daily_aggr_tblname` are referenced by downstream reporting jobs and possibly by other aggregation scripts that join on business‑trend data. |
| **Email/SDP ticketing** | The same email template and SDP ticket creation logic is reused across the Move‑Mediation suite (see `MNAAS_backup_SDP_ticket_validation.sh`). |
| **Scheduler** | Typically invoked by a cron entry or an Oozie/Airflow workflow that passes the month‑end date as `$1`. The script’s PID guard prevents overlapping runs, which is a pattern used by other scripts in the repository. |
| **Log aggregation** | Logs are written to a path that is likely collected by a central log‑management system (e.g., Splunk/ELK) used by the Move‑Mediation team. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale PID / false‑positive “already running”** | Add a health‑check that verifies the stored PID is still an active process belonging to this script; if not, clear the PID entry. |
| **Partial failure leaving Hive partitions inconsistent** | Ensure the partition‑drop step is idempotent; consider wrapping it in a Hive transaction or adding a “re‑run safe” flag. |
| **Log file growth** | Rotate the log file daily or when it exceeds a size threshold (logrotate). |
| **SDP ticket spamming on repeated failures** | The script already checks the `MNAAS_email_sdp_created` flag; verify that the flag is cleared only on successful run to avoid missing new tickets. |
| **Missing/incorrect properties** | Validate required variables at script start (e.g., `[[ -z $MNAAS_business_Trend_Data_OP_filename ]] && exit 1`). |
| **Java job memory/GC failures** | Monitor Java process exit codes and JVM logs; consider adding `-Xmx` tuning in the properties file. |
| **Impala metadata refresh failures** | Capture the exit status of `impala-shell`; if non‑zero, retry a limited number of times before aborting. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `MNAAS_business_trend_Aggr.properties` is present and readable.  
   - Verify that the process‑status file exists and is writable.  
   - Confirm HDFS, Hive, Impala, and Java environments are correctly configured for the user.

2. **Typical invocation** (from the scheduler or manually):  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_business_trend_Aggr.sh 2025-11-30
   ```

3. **Debug mode**  
   - The script already runs with `set -x`, which prints each command to the log.  
   - To see live output, tail the log: `tail -f $MNAAS_business_trend_AggrLogPath`.  
   - If a step fails, check the corresponding Java class logs (often written to the same log path) and the Hive/Impala error messages.

4. **Checking state**  
   - View the status file: `cat $MNAAS_business_trend_Aggr_ProcessStatusFileName`.  
   - The flag (`MNAAS_Daily_ProcessStatusFlag`) indicates the last completed step (0 = idle, 1‑4 = in‑progress).  
   - The PID field (`MNAAS_Script_Process_Id`) shows the running process; verify with `ps -p <PID>`.

5. **Force a restart** (e.g., after fixing a transient error)  
   - Ensure no process is running (`ps -p $(grep MNAAS_Script_Process_Id …)`).  
   - Reset the flag to `0` (or `1` to re‑run from the beginning) in the status file.  
   - Re‑execute the script with the desired date.

---

## 7. External Configurations & Environment Variables

| Variable (from properties) | Usage |
|----------------------------|-------|
| `MNAAS_business_trend_AggrLogPath` | Path to the script’s log file. |
| `MNAAS_business_trend_Aggr_ProcessStatusFileName` | Shared status/flag file. |
| `MNAAS_business_Trend_Data_OP_filename` | Local file where the raw aggregation is written. |
| `MNAAS_Daily_Aggregation_businesstrend_PathName` | HDFS staging directory. |
| `MNAAS_Main_JarPath`, `Generic_Jar_Names` | Classpath components for Java jobs. |
| `MNAAS_business_trend_aggregation_classname`, `MNAAS_business_trend_load_classname` | Fully‑qualified Java class names for the two aggregation steps. |
| `Dname_MNAAS_*` | Human‑readable names used for process‑level locking (`ps | grep`). |
| `drop_partition_in_table`, `Load_nonpart_table` | Java utility class names invoked for partition removal and loading. |
| `dbname`, `business_trend_daily_aggr_tblname`, `business_trend_inter_daily_aggr_tblname` | Hive database and table identifiers. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `business_trend_daily_aggr_tblname_refresh`, `business_trend_inter_daily_aggr_tblname_refresh` | Impala `REFRESH` statements (passed to `impala-shell`). |
| `MailId`, `ccList`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email addresses used for failure notifications. |
| `CLASSPATHVAR` | Additional classpath entries (often Hadoop/Hive jars). |
| `event_date` (script argument) | Date for which the aggregation is performed. |

---

## 8. Suggested Improvements (TODO)

1. **Add explicit configuration validation** – early in the script, loop over required variables and abort with a clear error if any are unset. This prevents obscure “command not found” failures later.
2. **Make the PID lock robust** – replace the simple `ps | grep` check with `flock` or a lock‑file mechanism that also verifies the stored PID is alive, reducing the chance of stale lock blocks. 

---