**MNAAS_Table_Space_Monitoring.sh – High‑Level Documentation**  
---  

### 1. Purpose (one‑paragraph summary)  
`MNAAS_Table_Space_Monitoring.sh` is a nightly housekeeping script that audits the HDFS storage consumption of a predefined set of Hive tables belonging to the **MNAAS** database. For each table it records the total table size, computes the average size of the last ten daily partitions, and calculates the deviation of yesterday’s partition from that average. The results are written to a dated CSV file, attached to an email sent to the operations and development mailing lists, and the script updates a shared *process‑status* file used by the broader “move‑mediation” orchestration framework to indicate success/failure and to prevent overlapping executions.  

---  

### 2. Key Functions / Logical Blocks  

| Function / Block | Responsibility |
|------------------|----------------|
| **`MNAAS_Space_Monitoring`** | Core routine: iterates over `table_list`, gathers HDFS size via `hadoop fs -du`, computes metrics, writes CSV, emails results, and sets success flags in the process‑status file. |
| **`terminateCron_successful_completion`** | Resets the daily‑process flag, records job status = Success, timestamps the run, logs termination, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Marks the job status as Failure in the status file, creates an SDP ticket flag (if not already set), and logs the action. |
| **Main Program (PID & flag handling)** | Prevents concurrent runs by checking a stored PID, updates the PID in the status file, validates the daily‑process flag (0 or 1), and dispatches the monitoring routine. |

---  

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Static Inputs** | • `MNAAS_Table_Space_Monitoring.properties` (sourced at start) – defines: <br>  `MNAAS_Table_Space_Monitoring_logpath` (log file), <br>  `MNAAS_Table_Space_Monitoring_ProcessStatusFilename` (shared status file), <br>  `MNAAS_Table_Space_Monitoring_Scriptname` (script identifier). |
| **Dynamic Inputs** | • HDFS paths: `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/<table>/partition_date=<date>` <br>• Current date / yesterday’s date (derived via `date`). |
| **Outputs** | • CSV file: `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/Space_Monitoring/Table_Space_Monitoring_YYYY-MM-DD.csv` (contains `Table_Name,Total_Table_Size_in_MB,Average_Size_of_Last_10_Partitions_in_KB,Deviation_of_Yesterday_Partition_From_Average_in_KB`). <br>• Email (via `mailx`) with CSV attached, sent to `move-mediation-it-ops@tatacommunications.com` and `hadoop-dev@tatacommunications.com` (CC list defined). |
| **Side‑effects** | • Appends diagnostic messages to `$MNAAS_Table_Space_Monitoring_logpath`. <br>• Updates the shared process‑status file (flags, PID, timestamps, job status). <br>• May trigger an SDP ticket creation flag on failure. |
| **External Services / Dependencies** | • Hadoop HDFS (`hadoop fs -du`). <br>• Local mail subsystem (`mailx`). <br>• System logger (`logger`). <br>• Underlying OS `ps` for PID check. |
| **Assumptions** | • All tables listed exist under the Hive warehouse path and are partitioned by `partition_date`. <br>• The user executing the script has read permission on HDFS paths and write permission on the log, CSV, and status files. <br>• `mailx` is correctly configured to send external mail. <br>• The properties file supplies all required variables; missing entries will cause script failure. |

---  

### 4. Interaction with Other Scripts / Components  

| Interaction | Description |
|-------------|-------------|
| **Process‑Status File** (`$MNAAS_Table_Space_Monitoring_ProcessStatusFilename`) | Shared across the *move‑mediation* suite (e.g., `MNAAS_Sim_Status_Monthly.sh`, `MNAAS_Sqoop_*.sh`). It stores flags such as `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, and the PID of the last run. Other jobs read these flags to decide whether to start or to wait. |
| **Cron Scheduler** | The script is intended to be invoked by a nightly cron entry (not shown). The PID‑guard prevents overlapping runs, a pattern used by other `MNAAS_*.sh` scripts. |
| **Logging Infrastructure** | All scripts in the suite write to a common log directory (`$MNAAS_Table_Space_Monitoring_logpath`). Ops monitor this location for alerts. |
| **Email Notification** | The same mailing list is used by other monitoring/orchestration scripts, providing a unified ops inbox. |
| **SDP Ticketing** | Failure handling sets a flag (`MNAAS_email_sdp_created`) that downstream ticket‑generation processes (outside this script) watch to raise an incident. |

---  

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **HDFS Path / Table Changes** | `hadoop fs -du` fails → script aborts, no email/report. | • Maintain a version‑controlled table list. <br>• Add existence checks (`hdfs dfs -test -e`) before `du`. |
| **Permission Issues** (HDFS, local files) | Log/CSV cannot be written → silent failures. | • Run script under a dedicated service account with explicit ACLs. <br>• Verify permissions during deployment. |
| **Email Delivery Failure** | Ops not notified → issue may go unnoticed. | • Capture `mailx` exit code; on non‑zero, log and raise SDP ticket. |
| **Concurrent Execution** (PID file stale) | Duplicate runs cause race conditions on status file. | • Validate that stored PID is still alive (`ps -p $PID`). <br>• Clean stale PID on system reboot. |
| **Large Table Sizes** (performance) | Script runtime exceeds cron window. | • Parallelize per‑table processing (background subshells) or limit to critical tables. |
| **Missing Properties** | Undefined variables cause runtime errors. | • Add sanity check after sourcing properties (e.g., `[[ -z $var ]] && exit 1`). |

---  

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|------------------|
| **Manual execution** | `bash /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Table_Space_Monitoring.sh` |
| **Enable verbose tracing** (already set with `set -x`) – useful for debugging. |
| **Check logs** | `tail -f $MNAAS_Table_Space_Monitoring_logpath` |
| **Validate PID guard** | `cat $MNAAS_Table_Space_Monitoring_ProcessStatusFilename` → look for `MNAAS_Script_Process_Id`. |
| **Force a run ignoring PID** | `export FORCE_RUN=1` (add a conditional in the script) or manually delete the PID entry. |
| **Test email** | Replace `$MOVE_DEV_TEAM` with a test address, run script, verify receipt. |
| **Dry‑run HDFS size queries** | `hadoop fs -du -s hdfs://NN-HA1/user/hive/warehouse/mnaas.db/<table>` |
| **Inspect generated CSV** | `cat /app/hadoop_users/MNAAS/MNAAS_Configuration_Files/Space_Monitoring/Table_Space_Monitoring_$(date +%F).csv` |

---  

### 7. External Configuration & Environment Variables  

| Variable (from properties) | Role |
|----------------------------|------|
| `MNAAS_Table_Space_Monitoring_logpath` | Full path to the script‑specific log file. |
| `MNAAS_Table_Space_Monitoring_ProcessStatusFilename` | Shared status file that holds flags, PID, timestamps. |
| `MNAAS_Table_Space_Monitoring_Scriptname` | Identifier used in status file and log messages. |
| `MOVE_DEV_TEAM` | Comma‑separated list of primary recipients for the email (defined elsewhere). |
| `mail_from` / `mail_to` / `cc_List` | Hard‑coded in script; used by `mailx`. |

If any of these are missing or point to non‑existent locations, the script will abort; therefore the properties file must be validated during deployment.  

---  

### 8. Suggested Improvements (TODO)  

1. **Add Robust Error Handling for HDFS Commands** – capture non‑zero exit codes from `hadoop fs -du`, log a clear error, and continue to next table rather than aborting the whole run.  
2. **Externalize Table List** – move `table_list` into a configuration file (e.g., `MNAAS_Table_Space_Monitoring.tables`) so that adding/removing tables does not require script modification and can be version‑controlled separately.  

---  

*End of documentation.*