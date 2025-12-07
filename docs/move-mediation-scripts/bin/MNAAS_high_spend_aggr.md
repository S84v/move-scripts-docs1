**MNAAS_high_spend_aggr.sh – High‑Spend Monthly Usage Trend Aggregation**

---

### 1. High‑Level Summary
`MNAAS_high_spend_aggr.sh` is a Bash driver that builds the “high‑spend” monthly usage‑trend table used by the Move mediation platform. It runs once per day (via cron) and orchestrates four logical steps:

1. **File creation** – extracts raw high‑spend CDR data into a flat file using a Java aggregation job.  
2. **Partition cleanup** – drops yesterday’s Hive/Impala partitions for the target table.  
3. **Intermediate load** – copies the flat file to HDFS, then loads it into a temporary non‑partitioned Hive table via a Java loader.  
4. **Final aggregation load** – moves data from the temporary table into the partitioned “high‑spend daily aggregation” table and refreshes Impala metadata.

The script maintains a *process‑status* file that records the current step (flag 0‑4), PID, and overall job status. On any failure it logs the error, sends an email, and raises an SDP ticket; on success it updates the status file to “Success”.

---

### 2. Important Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_creation_of_file** | Runs the Java class `$MNAAS_high_spend_aggregation_classname` to generate `$MNAAS_high_spend_Data_filename`. Updates status flag 1. |
| **MNAAS_remove_partitions_from_the_table** | Executes a Java utility (`$drop_partitions_in_table`) to drop yesterday’s partitions from `$high_spend_daily_aggr_tblname`. Refreshes Impala metadata. Updates status flag 2. |
| **MNAAS_high_spend_inter_table_loading** | Clears HDFS target dir, copies the generated flat file, sets permissions, then runs `$Load_nonpart_table` Java loader to populate a temporary table `$high_spend_inter_daily_aggr_tblname`. Updates status flag 3. |
| **MNAAS_high_spend_aggr_table_loading** | Calls `$MNAAS_high_spend_load_classname` Java job to move data from the temporary table into the final partitioned table. Refreshes Impala metadata. Updates status flag 4. |
| **terminateCron_successful_completion** | Resets status flag 0, marks job as *Success*, records run‑time, writes final log entries and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Updates status to *Failure*, checks if an email/SDP ticket has already been sent, then sends a notification (mail/mailx) and creates an SDP ticket via a templated email to `insdp@tatacommunications.com`. |
| **Main block** | PID guard (prevents concurrent runs), reads current flag from the status file, and dispatches the appropriate subset of functions to allow restart from the last successful step. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_high_spend_aggr.properties` – defines all environment variables used throughout the script (paths, class names, DB/table names, hosts, ports, email lists, log files, etc.). |
| **Process‑status file** | `$MNAAS_high_spend_Aggr_ProcessStatusFileName` – plain‑text key/value file storing flags, PID, job status, timestamps, email‑sent flag. Updated by every function. |
| **Log file** | `$MNAAS_high_spend_AggrLogPath` – appended with `logger -s` statements; also receives stderr (`exec 2>>`). |
| **Generated data file** | `$MNAAS_high_spend_Data_filename` – flat file created by the first Java job; later copied to HDFS. |
| **HDFS directory** | `$MNAAS_Daily_Aggregation_high_spend_PathName` – temporary staging area for the flat file before loading. |
| **Hive/Impala tables** | `$high_spend_daily_aggr_tblname` (partitioned target) and `$high_spend_inter_daily_aggr_tblname` (temporary non‑partitioned). |
| **External services** | - Java runtime (JARs on `$CLASSPATHVAR`, `$Generic_Jar_Names`, `$MNAAS_Main_JarPath`).<br>- Hadoop HDFS (`hadoop fs`, `hdfs dfs`).<br>- Hive/Impala (`impala-shell`).<br>- Mail system (`mail`, `mailx`).<br>- SDP ticketing (via email to `insdp@tatacommunications.com`). |
| **Side effects** | - Creation/modification of HDFS files.<br>- Hive/Impala DDL (drop partitions, refresh metadata).<br>- Sending of email alerts and SDP tickets.<br>- Updating of shared status file used by other Move scripts. |
| **Assumptions** | - All referenced environment variables are defined in the properties file.<br>- Required Java classes are present and compatible with the current Hadoop/Hive versions.<br>- The script runs on a host with Hadoop client, Impala client, and mail utilities installed.<br>- The status file is writable by the script user and not concurrently edited by other processes. |

---

### 4. Connections to Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **Cron scheduler** | The script is invoked by a daily cron entry (e.g., `0 2 * * * /path/MNAAS_high_spend_aggr.sh`). |
| **Other Move “high‑spend” scripts** | The status file (`*_ProcessStatusFileName`) is shared with any recovery or monitoring scripts that may query the flag to decide whether to re‑run or alert. |
| **MNAAS_edgenode* backup scripts** | Not a direct call, but they use the same property‑file pattern and may rely on the same JARs (`$Generic_Jar_Names`). |
| **SDP ticketing system** | Email sent by `email_and_SDP_ticket_triggering_step` creates a ticket; downstream ticket‑handling processes consume it. |
| **Monitoring dashboards** | The status file fields (`MNAAS_job_status`, `MNAAS_job_ran_time`) are likely scraped by Ops monitoring tools to display job health. |
| **Java aggregation classes** | `MNAAS_high_spend_aggregation_classname`, `drop_partitions_in_table`, `Load_nonpart_table`, `MNAAS_high_spend_load_classname` – these are compiled components shared across the Move platform. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID guard may miss a stale PID (process died unexpectedly). | Duplicate loads, data corruption. | Add a health‑check (e.g., verify the PID still exists and is the same script) and a timeout fallback that clears stale PID entries. |
| **Java job failure** – non‑zero exit code not captured if the Java process forks. | Silent data loss, downstream tables empty. | Ensure Java wrappers exit with the correct status; capture stdout/stderr to log files for post‑mortem. |
| **HDFS permission errors** – `chmod 777` may be insufficient on secure clusters. | Load step aborts. | Use proper HDFS ACLs; log the exact `hdfs dfs -chmod` error. |
| **Impala metadata refresh failure** – `impala-shell` may time out. | Queries see stale data. | Retry the refresh a configurable number of times; log each attempt. |
| **Email/SDP ticket flood** – repeated failures could generate many tickets. | Ops overload. | Add a throttling flag (e.g., only send ticket if last ticket > 4 h ago) and include a “re‑trigger” counter in the status file. |
| **Properties file drift** – missing or malformed variables cause script aborts. | Immediate failure. | Validate required variables at script start; abort with clear error if any are undefined. |
| **Log file growth** – unbounded appends may fill disk. | Cron job stops. | Rotate logs via logrotate or implement size‑based truncation within the script. |

---

### 6. Running / Debugging the Script

| Step | Command / Action |
|------|-------------------|
| **Normal execution** | `bash /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_high_spend_aggr.sh` (usually invoked by cron). |
| **Force a fresh run** | Delete or reset the status file flag to `0` (e.g., `sed -i 's/MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=0/' $MNAAS_high_spend_Aggr_ProcessStatusFileName`). |
| **Run a single step** | Call the function directly after sourcing the script: `source MNAAS_high_spend_aggr.sh && MNAAS_creation_of_file`. |
| **Enable verbose tracing** | The script already uses `set -x`; you can also export `BASH_XTRACEFD=5` and redirect to a separate file if needed. |
| **Inspect status** | `cat $MNAAS_high_spend_Aggr_ProcessStatusFileName` – look at `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_Script_Process_Id`. |
| **Check logs** | `tail -f $MNAAS_high_spend_AggrLogPath`. |
| **Debug Java jobs** | Add `-verbose:gc` or other JVM flags in the property file; capture stdout/stderr to separate files. |
| **Validate HDFS** | `hdfs dfs -ls $MNAAS_Daily_Aggregation_high_spend_PathName` before and after the load step. |
| **Test email/SDP** | Temporarily set `MNAAS_email_sdp_created=No` and run a failure path to confirm mail delivery. |

---

### 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_high_spend_AggrLogPath` | Path to the script’s log file. |
| `MNAAS_high_spend_Aggr_ProcessStatusFileName` | Shared status file. |
| `MNAAS_high_spend_Data_filename` | Local flat file generated by the aggregation Java job. |
| `MNAAS_Daily_Aggregation_high_spend_PathName` | HDFS staging directory. |
| `MNAAS_high_spend_aggregation_classname` | Fully‑qualified Java class for step 1. |
| `MNAAS_high_spend_load_classname` | Java class for final load (step 4). |
| `drop_partitions_in_table` | Java class for partition removal (step 2). |
| `Load_nonpart_table` | Java class for loading temp table (step 3). |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | JAR locations for the Java jobs. |
| `dbname`, `high_spend_daily_aggr_tblname`, `high_spend_inter_daily_aggr_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive metastore connection. |
| `IMPALAD_HOST` | Impala daemon for metadata refresh. |
| `high_spend_daily_aggr_tblname_refresh` | Impala `REFRESH` statement string. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / headers for failure notifications. |
| `MNAAS_high_spend_aggr_ScriptName` | Human‑readable script identifier used in logs/emails. |
| `MNAAS_Script_Process_Id` | PID stored in the status file. |

*All of the above are defined in `MNAAS_high_spend_aggr.properties`. The script assumes the file is present and readable.*

---

### 8. Suggested Improvements (TODO)

1. **Parameter Validation Block** – Add a function at the top that checks every required variable from the properties file (e.g., `[[ -z $MNAAS_high_spend_Data_filename ]] && echo "Missing var" && exit 1`). This prevents silent failures when a property is missing or mistyped.

2. **Retry Logic for External Calls** – Wrap the `impala-shell` refresh and Hadoop `copyFromLocal` commands in a small retry loop (max 3 attempts with exponential back‑off). This will make the job more resilient to transient HDFS/Impala hiccups.