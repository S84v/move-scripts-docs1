**MNAAS_Traffic_Adhoc_Aggr.sh – High‑Level Documentation**  
---  

### 1. Purpose (one‑paragraph summary)  
`MNAAS_Traffic_Adhoc_Aggr.sh` orchestrates the ad‑hoc aggregation of the **MNAAS traffic usage** data set. It is invoked (typically by a cron job) after raw traffic files have been loaded into staging tables. The script removes stale Hive partitions, runs two Java‑based aggregation steps (inter‑mediate table load and final table load), refreshes the corresponding Impala metadata, and maintains a process‑status file that drives incremental execution and error handling. On failure it creates an SDP ticket and notifies the Move‑Dev team via email.  

---  

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_remove_partitions_from_the_table** | Updates the status file, sets HDFS permissions, launches the Java *drop_partitions_in_table* utility to delete old Hive partitions for the ad‑hoc traffic aggregation table, then refreshes Impala metadata. |
| **MNAAS_traffic_aggr_adhoc_inter_table_loading** | Marks step 2 in the status file, runs the Java *traffic_aggr_adhoc_aggregation* class to populate an intermediate aggregation table, and refreshes Impala. |
| **MNAAS_traffic_aggr_adhoc_table_loading** | Marks step 3, starts a background refresh script (`traffic_aggr_adhoc_refresh.sh`), runs the Java *MNAAS_traffic_aggr_adhoc_load* class to load the final aggregated table, and refreshes Impala. |
| **terminateCron_successful_completion** | Writes a “Success” flag, timestamps, and a clean‑exit message to the status file and log, then exits 0. |
| **terminateCron_Unsuccessful_completion** | Writes a “Failure” flag, triggers ticket/email creation, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends an SDP ticket via `mailx` (if not already sent) with details of the failure, updates the status file to indicate ticket creation, and logs the action. |

---  

### 3. Inputs, Outputs & Side Effects  

| Category | Item | Description |
|----------|------|-------------|
| **Configuration** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Traffic_Adhoc_Aggr.properties` | Bash source file that defines all environment variables used throughout the script (paths, table names, Java class names, Hive/Impala hosts, etc.). |
| **Process‑status file** | `$MNAAS_traffic_aggr_adhoc_ProcessStatusFileName` | Plain‑text key/value file that stores flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_Script_Process_Id`, etc.). Used for concurrency control and step‑wise restart. |
| **Log file** | `$MNAAS_traffic_aggr_adhocLogPath` | Append‑only log where all `logger` output and script diagnostics are written. |
| **Java JARs / Classes** | `$CLASSPATHVAR`, `$Generic_Jar_Names`, `$MNAAS_Main_JarPath` | Classpath for the three Java utilities invoked (drop partitions, aggregation, final load). |
| **HDFS / Hive** | `/user/hive/warehouse/mnaas.db/traffic_aggr_adhoc/*` | Partition directories that may be removed or refreshed. |
| **Impala** | `$IMPALAD_HOST` + refresh queries (`${traffic_aggr_adhoc_tblname_refresh}`, `${traffic_aggr_adhoc_inter_tblname_refresh}`) | Metadata refresh after each load step. |
| **External script** | `/app/hadoop_users/MNAAS/MNAAS_CronFiles/traffic_aggr_adhoc_refresh.sh` | Launched in background by the final load step. |
| **Email / SDP** | `mailx` with `$SDP_ticket_from_email`, `$SDP_ticket_to_email`, `$MOVE_DEV_TEAM` | Sent on failure (once per run). |
| **Outputs** | Updated Hive/Impala tables (`traffic_aggr_adhoc_tblname`, `traffic_aggr_adhoc_inter_tblname`) | Final aggregated data ready for downstream reporting. |
| **Side effects** | Process‑status file mutation, log growth, possible ticket creation, Impala metadata refresh, HDFS permission changes. |
| **Assumptions** | • All required Java JARs are present and compatible.<br>• Hive/Impala services are reachable (`$HIVE_HOST`, `$IMPALAD_HOST`).<br>• The status file exists and is writable.<br>• `mailx` is configured for the internal mail system.<br>• No other instance of the script (or its Java jobs) is running concurrently. |

---  

### 4. Interaction with Other Scripts / Components  

| Component | Relationship |
|-----------|--------------|
| **MNAAS_TrafficDetails_* scripts** | Load raw traffic files into staging tables that feed the ad‑hoc aggregation tables used here. |
| **MNAAS_Traffic_Adhoc_Aggr.properties** | Centralised configuration shared with other traffic‑related scripts (e.g., daily aggregation, backup). |
| **`traffic_aggr_adhoc_refresh.sh`** | Triggered by `MNAAS_traffic_aggr_adhoc_table_loading` to perform any post‑load housekeeping (e.g., cache invalidation). |
| **Cron scheduler** | Typically registers this script as a daily/weekly ad‑hoc job; the script itself checks for an existing PID to avoid overlap. |
| **SDP ticketing system** | Integrated via email; failures are reported to the support queue (`Cloudera.Support@tatacommunications.com`). |
| **Impala / Hive Metastore** | Refresh queries are executed after each load to make new partitions visible to downstream queries. |
| **Java utilities** (`drop_partitions_in_table`, `traffic_aggr_adhoc_aggregation_classname`, `MNAAS_traffic_aggr_adhoc_load_classname`) | Core data‑movement logic; they are invoked with the same property file and partition‑list file. |
| **Process‑status file** | Shared with other MNAAS scripts to coordinate multi‑step pipelines (e.g., daily vs. ad‑hoc runs). |

---  

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (multiple instances or overlapping Java jobs) | Data corruption, duplicate loads, lock contention. | The script checks the PID in the status file; enforce a filesystem lock (e.g., `flock`) and verify the Java process list more robustly (`pgrep -f`). |
| **Java job failure (non‑zero exit)** | Incomplete aggregation, stale partitions. | All Java calls are wrapped with exit‑code checks; add retry logic or fallback to a “manual re‑run” flag in the status file. |
| **Impala refresh failure** | New partitions not visible → downstream reports missing data. | Capture `impala-shell` exit status; if non‑zero, log and trigger a separate alert (e.g., Nagios). |
| **Missing or malformed properties file** | Script aborts early, no logs. | Validate required variables at start (`[[ -z $var ]] && echo "Missing $var" && exit 1`). |
| **Email/SDP ticket not sent** | Failure goes unnoticed. | Verify `mailx` return code; optionally duplicate alert to a monitoring system (e.g., PagerDuty). |
| **Log file growth** | Disk exhaustion on the node. | Rotate logs via `logrotate` or implement size‑based truncation in the script. |
| **HDFS permission change (`chmod 777`)** | Security exposure. | Restrict to required users/groups; replace with ACLs if possible. |

---  

### 6. Running / Debugging the Script  

1. **Prerequisites** – Ensure the properties file exists and all environment variables it defines are exported. Verify that the status file path is writable.  
2. **Manual execution** –  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_CronFiles   # or wherever the script resides
   ./MNAAS_Traffic_Adhoc_Aggr.sh
   ```  
   The script already runs with `set -x`, so each command is echoed to the log (`$MNAAS_traffic_aggr_adhocLogPath`).  
3. **Check PID lock** – If you see “Previous traffic adhoc aggr Process is running already”, inspect the status file:  
   ```bash
   cat $MNAAS_traffic_aggr_adhoc_ProcessStatusFileName | grep MNAAS_Script_Process_Id
   ps -fp <pid>
   ```  
   Kill stale PID if necessary (after confirming no legitimate run).  
4. **Force a specific step** – Edit the status file’s `MNAAS_Daily_ProcessStatusFlag` to 1, 2, or 3 to start from a particular stage (useful for recovery).  
5. **Log inspection** – Tail the log while the script runs:  
   ```bash
   tail -f $MNAAS_traffic_aggr_adhocLogPath
   ```  
6. **Java job debugging** – The Java utilities write their own logs (paths are passed as arguments). Review those if the script reports “failed”.  
7. **Impala validation** – After a successful run, run a quick query:  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "SELECT count(*) FROM $traffic_aggr_adhoc_tblname LIMIT 1;"
   ```  

---  

### 7. External Config / Environment Variables  

| Variable (defined in properties) | Role |
|----------------------------------|------|
| `MNAAS_traffic_aggr_adhocLogPath` | Absolute path to the script’s log file. |
| `MNAAS_traffic_aggr_adhoc_ProcessStatusFileName` | Path to the shared status file. |
| `MNAAS_Traffic_AdhocPartitions_Daily_Uniq_FileName` | File containing the list of partitions to drop. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components for the three Java utilities. |
| `drop_partitions_in_table` | Fully‑qualified Java class name for partition removal. |
| `traffic_aggr_adhoc_aggregation_classname` | Java class for the intermediate aggregation step. |
| `MNAAS_traffic_aggr_adhoc_load_classname` | Java class for the final load step. |
| `traffic_aggr_adhoc_tblname`, `traffic_aggr_adhoc_inter_tblname` | Hive/Impala table names. |
| `traffic_aggr_adhoc_tblname_refresh`, `traffic_aggr_adhoc_inter_tblname_refresh` | Impala `REFRESH` statements (e.g., `REFRESH $traffic_aggr_adhoc_tblname`). |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Service endpoints. |
| `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM` | Email routing for failure tickets. |
| `Dname_MNAAS_drop_partitions_traffic_aggr_adhoc_tbl`, `Dname_MNAAS_Load_Daily_traffic_aggr_adhoc_daily_aggr_temp`, `Dname_MNAAS_traffic_aggr_adhoc_load` | Unique identifiers used to detect already‑running Java processes. |
| `MNAAS_traffic_aggr_adhoc_ScriptName` | Human‑readable script identifier used in logs and tickets. |

---  

### 8. Suggested Improvements (TODO)  

1. **Replace ad‑hoc PID lock with `flock`** – Guarantees atomic lock acquisition and eliminates race conditions when multiple cron instances start simultaneously.  
2. **Externalize Impala refresh statements** – Store them in the properties file and validate their syntax before execution; this makes the script easier to adapt to schema changes without code edits.  

---  

*End of documentation.*