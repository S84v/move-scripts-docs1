**MNAAS_TrafficDetails_hourly_tbl_aggr.sh – High‑Level Documentation**  
---  

### 1. Purpose (one‑paragraph summary)  
This script performs the hourly aggregation of raw traffic‑detail partitions for the MNAAS mediation layer. It deduplicates the incoming partition list, drops existing hourly partitions from the target Hive/Impala aggregation table, recreates them via a temporary “intermediate” table, loads the cleaned data into the main aggregation table, copies the processed partition list for the downstream daily aggregation, and finally triggers the daily‑aggregation driver. The script is state‑driven – a process‑status file records the last successful step so that a failed run can be resumed without re‑processing already‑completed stages. It is typically invoked by a scheduler (cron) with the current hour and minute as arguments.  

---  

### 2. Key Functions / Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **MNAAS_remove_dups_from_the_file** | Sorts the raw hourly partition file, removes duplicate lines, strips any `WARN` entries, and updates the process‑status flag to **1**. |
| **MNAAS_remove_partitions_from_the_table** | Executes a Java utility (`drop_partitions_in_table`) to drop the identified hourly partitions from the Hive aggregation table, refreshes the Impala metadata, and sets flag **2**. |
| **MNAAS_recreate_partitions_in_the_inter_table** | Runs a Java utility (`traffic_aggr_dups_remove`) that rebuilds the partitions in a temporary/intermediate table, flag set to **3**. |
| **MNAAS_load_into_main_table** | Calls a Java utility (`Load_traffic_aggr_table`) to move data from the intermediate table into the final hourly aggregation table, refreshes Impala, flag set to **4**. |
| **MNAAS_take_the_copy_of_file_for_daily_aggr** | Appends the cleaned hourly partition list to the daily‑aggregation file, flag set to **5**. |
| **MNAAS_remove_the_contents_of_the_file** | Truncates the raw hourly partition file to prepare for the next hour, flag set to **6**. |
| **MNAAS_daily_aggr_script** | Invokes the daily‑aggregation driver script (`$MNAASDailyTrafficDetails_daily_Aggr_SriptName`) with the current hour/minute, flag set to **7**. |
| **terminateCron_successful_completion** | Resets the status file to “Success”, clears the flag, logs completion, and exits `0`. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers email/SDP ticket, and exits `1`. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email and creates an SDP ticket (via `mailx`) if not already sent. |

---  

### 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • Command‑line arguments: `$1` = hour, `$2` = minute.<br>• Property file: `MNAAS_TrafficDetails_hourly_tbl_aggr.properties` (defines all environment variables).<br>• Files on local FS (or HDFS mounted locally):<br> - `$MNAAS_Traffic_Partitions_Hourly_FileName` (raw partition list).<br> - `$MNAAS_Daily_Traffictable_Load_Hourly_Aggr_ProcessStatusFileName` (state file). |
| **Outputs** | • Log file: `$MNAAS_DailyTrafficDetailsLoad_Hourly_AggrLogPath` (stderr redirected).<br>• Updated process‑status file (flags, timestamps, PID, job status).<br>• Updated hourly partition file (`*_Uniq_FileName`).<br>• Updated daily‑aggregation partition file (`$MNAAS_Traffic_Partitions_Daily_FileName`). |
| **Side‑effects** | • Hive/Impala tables are altered (partitions dropped/added, data loaded).<br>• Java processes are launched (requires Java runtime, classpath, JARs).<br>• System mail is sent on failure; SDP ticket created via `mailx`.<br>• PID stored in status file to prevent concurrent runs. |
| **Assumptions** | • All variables referenced are defined in the sourced `.properties` file.<br>• Java classes (`drop_partitions_in_table`, `traffic_aggr_dups_remove`, `Load_traffic_aggr_table`) are present and compatible with the supplied arguments.<br>• Hive Metastore and Impala daemons are reachable (`$HIVE_HOST`, `$IMPALAD_HOST`).<br>• The script runs on a host with required permissions to read/write the files, execute Java, and run `impala-shell`. |
| **External Services** | • Hive/Impala (SQL engines).<br>• SMTP server for `mail`/`mailx`.<br>• SDP ticketing system (email‑based).<br>• Possibly HDFS (if the partition files reside there). |

---  

### 4. Integration Points (how this script connects to other components)  

| Connected Component | Interaction |
|---------------------|-------------|
| **MNAAS_TrafficDetails_hourly_tbl_aggr.properties** | Provides all configurable paths, DB names, hostnames, classpath, JAR locations, and the names of the Java classes invoked. |
| **Java utilities** (`drop_partitions_in_table`, `traffic_aggr_dups_remove`, `Load_traffic_aggr_table`) | Called via `java -cp …` to manipulate Hive tables. |
| **Impala‑shell** | Used after partition drop and after final load to refresh table metadata (`${aggr_traffic_details_hourly_tblname_refresh}`). |
| **MNAASDailyTrafficDetails_daily_Aggr_SriptName** (daily aggregation driver) | Triggered by `MNAAS_daily_aggr_script` to roll up hourly data into daily aggregates. |
| **Process‑status file** (`$MNAAS_Daily_Traffictable_Load_Hourly_Aggr_ProcessStatusFileName`) | Shared with other scripts (e.g., daily‑aggregation, other hourly loaders) to coordinate restart/recovery. |
| **Logging & Alerting** | Writes to `$MNAAS_DailyTrafficDetailsLoad_Hourly_AggrLogPath`; on failure sends email to `$GTPMailId` and SDP ticket to `insdp@tatacommunications.com`. |
| **Cron / Scheduler** | Typically invoked by a cron entry that passes the current hour/minute. |
| **Potential downstream** | The daily‑aggregation script may subsequently invoke other “load” scripts (e.g., `MNAAS_TrafficDetails_daily_tbl_aggr.sh`). |

---  

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID / orphaned lock** | Subsequent runs are blocked, causing data backlog. | Add a timeout check on the stored PID (e.g., verify process age) and allow manual reset of the status file. |
| **Partial partition drop** | Data loss if drop succeeds but load fails. | Ensure the flag‑based resume logic only proceeds after successful load; consider using Hive “transactional” tables or backup snapshots before drop. |
| **Java class failure (class not found, OOM)** | Script aborts, leaving tables in inconsistent state. | Validate classpath and JAR availability at start; monitor Java exit codes; add memory limits. |
| **Mail/SDP flood on repeated failures** | Alert fatigue. | De‑duplicate alerts by checking `MNAAS_email_sdp_created` flag (already done) and add rate‑limiting. |
| **Incorrect property values (e.g., wrong table name)** | Wrong tables are modified. | Include a pre‑run validation step that checks existence of target tables and logs a clear error before any DDL. |
| **File permission issues** | Unable to read/write partition files or status file. | Run script under a dedicated service account with explicit ACLs; verify permissions at start. |
| **Concurrent runs on different hosts** | Duplicate processing. | Centralise the PID lock in a shared location (e.g., HDFS lock file) or use a distributed lock service. |

---  

### 6. Typical Execution / Debugging Steps  

1. **Invocation** (usually from cron):  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_TrafficDetails_hourly_tbl_aggr.sh 14 05
   ```
   – Passes hour `14` and minute `05`.  

2. **Check prerequisites** before running:  
   - Verify the properties file exists and is readable.  
   - Ensure `$MNAAS_Traffic_Partitions_Hourly_FileName` is present and non‑empty.  
   - Confirm Java, Hive, Impala, and mail services are reachable.  

3. **Monitoring** while running:  
   - Tail the log: `tail -f $MNAAS_DailyTrafficDetailsLoad_Hourly_AggrLogPath`.  
   - Observe the process‑status file for flag changes (`grep MNAAS_Daily_ProcessStatusFlag …`).  

4. **Debugging failures**:  
   - The script already runs with `set -x` (command tracing).  
   - If a step fails, the log will contain “failed” and the script will call `terminateCron_Unsuccessful_completion`.  
   - Examine the exit code of the Java command (`echo $?`) and the corresponding Java logs (if any).  
   - Verify that the PID stored in the status file matches a running process (`ps -p <pid>`).  

5. **Manual recovery** (after a failure):  
   - Edit the status file to set `MNAAS_Daily_ProcessStatusFlag` to the last successful step (e.g., `3` to resume from recreation).  
   - Re‑run the script with the same hour/minute arguments.  

---  

### 7. External Configuration / Environment Variables  

| Variable (from properties) | Role |
|----------------------------|------|
| `MNAAS_DailyTrafficDetailsLoad_Hourly_AggrLogPath` | Path for stderr redirection and logger output. |
| `MNAAS_Daily_Traffictable_Load_Hourly_Aggr_ProcessStatusFileName` | Central state file (flags, PID, timestamps). |
| `MNAAS_Traffic_Partitions_Hourly_FileName` | Raw partition list (input). |
| `MNAAS_Traffic_Partitions_Hourly_Uniq_FileName` | Deduplicated partition list (intermediate). |
| `MNAAS_Traffic_Partitions_Daily_FileName` | Accumulated daily partition list (output for daily aggregation). |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath and JAR locations. |
| `drop_partitions_in_table`, `traffic_aggr_dups_remove`, `Load_traffic_aggr_table` | Fully‑qualified Java class names invoked. |
| `dbname`, `traffic_details_aggr_hourly_tblname`, `aggr_traffic_details_hourly_tblname_refresh` | Hive/Impala database and table identifiers. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive and Impala. |
| `MNAASDailyTrafficDetails_daily_Aggr_SriptName` | Name of the downstream daily‑aggregation driver script. |
| `GTPMailId`, `ccList`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients for failure notifications. |
| `hour_var`, `min_var` | Expected hour/minute values (used for time‑window gating). |
| `MNAASShellScriptPath` | Directory containing the daily‑aggregation script. |

---  

### 8. Suggested Improvements (TODO)  

1. **Refactor repetitive `sed` flag updates** into a helper function, e.g., `set_process_flag <num> <name>`, to reduce copy‑paste errors and simplify future flag changes.  
2. **Introduce a centralized lock service** (e.g., Zookeeper or a shared HDFS lock file) instead of a PID stored in a local file, eliminating the risk of stale locks when the script is run on multiple nodes.  

---  

*End of documentation.*