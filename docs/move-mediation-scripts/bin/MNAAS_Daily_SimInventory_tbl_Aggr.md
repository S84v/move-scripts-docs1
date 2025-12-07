**File:** `move-mediation-scripts\bin\MNAAS_Daily_SimInventory_tbl_Aggr.sh`  

---

## 1. High‑Level Summary
This Bash script is the daily driver for the **SimInventory aggregation** process in the MNAAS (Move‑Mediation‑NAA‑S) data‑move platform. It coordinates a Java aggregation job, refreshes the resulting Impala table, updates a shared process‑status file, writes detailed log entries, and on failure raises an SDP ticket and sends an email notification. The script is intended to be invoked by a scheduler (e.g., cron) and includes safeguards to prevent concurrent executions.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_insert_into_aggr_table`** | *Updates* the process‑status flag to “running”, checks that the Java aggregation job is not already running, launches the Java class `Insert_Daily_Aggr_table` with required arguments, validates its exit code, triggers an Impala `REFRESH` on the daily aggregation table, and logs success or failure. |
| **`terminateCron_successful_completion`** | *Marks* the job as successful in the status file, records the run timestamp, writes final log entries, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | *Logs* failure, calls `email_and_SDP_ticket_triggering_step`, and exits with a non‑zero status. |
| **`email_and_SDP_ticket_triggering_step`** | *Sets* job status to “Failure”, checks the flag `MNAAS_email_sdp_created`; if no ticket/email has been sent, it sends a formatted email to the MOVE dev team and creates an SDP ticket via `mailx`, then updates the status file to indicate that notification has been issued. |
| **Main script block** | *Ensures* only one instance of the script runs (process‑count guard), reads the current process flag from the shared status file, decides whether to start the aggregation (flags 0 or 1) or abort, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`. Expected variables include (but are not limited to):<br>• `MNAAS_Daily_SimInventory_Aggr_ProcessStatusFileName` (path to status file)<br>• `Dname_MNAAS_Insert_Daily_Aggr_SimInventory_tbl` (process name for Java job)<br>• `CLASSPATHVAR`, `MNAAS_Main_JarPath` (Java classpath & JAR location)<br>• `Insert_Daily_Aggr_table` (Java main class name)<br>• `MNAAS_Property_filename`, `MNAAS_Daily_SimInventory_AggrCntrlFileName`, `processname_siminventory` (control parameters for Java job)<br>• `MNAAS_DailySimInventoryAggrLogPath` (directory for daily logs)<br>• `IMPALAD_HOST`, `aggr_siminventory_daily_tblname_refresh` (Impala host & refresh query)<br>• Email‑related vars: `ccList`, `MailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` |
| **External Services** | • **Java runtime** – runs the aggregation JAR.<br>• **Impala** – `impala-shell` used to refresh the target table.<br>• **Mail / mailx** – sends failure notification and creates an SDP ticket.<br>• **File system** – reads/writes the status file and daily log files. |
| **Primary Outputs** | • Log file: `$MNAAS_DailySimInventoryAggrLogPath$(date +_%F)` (appended throughout the run).<br>• Updated status file (`MNAAS_Daily_SimInventory_Aggr_ProcessStatusFileName`) with flags: `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_job_ran_time`, etc.<br>• Impala table refresh (no direct file output). |
| **Side‑Effects** | • Potential SDP ticket creation (via `mailx`).<br>• Email sent to MOVE dev team and cc list.<br>• Java process may generate its own intermediate files (outside script scope). |
| **Assumptions** | • All variables referenced in the properties file are defined and point to accessible resources.<br>• The Java class `Insert_Daily_Aggr_table` is idempotent for a given day.<br>• Impala host is reachable and the refresh query is valid.<br>• Mail utilities (`mail`, `mailx`) are correctly configured on the host.<br>• The status file exists and is writable by the script user. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | How this script interacts |
|---------------------|---------------------------|
| **MNAAS_Daily_SimInventory_Load.sh** (hypothetical) | Likely loads raw SimInventory data into a staging table; this aggregation script runs **after** that load completes, using the same status file mechanism. |
| **MNAAS_Daily_SimInventory_tbl_Aggr_previous.sh** | May contain legacy logic; the current script supersedes it but shares the same status file and log path. |
| **Cron scheduler** | The script is scheduled (e.g., `0 2 * * *`) and relies on the process‑count guard (`ps aux | grep -w $MNAASDailySimInventoryAggrScriptName`) to avoid overlapping runs. |
| **Impala Metastore** | The `impala-shell -i $IMPALAD_HOST -q "${aggr_siminventory_daily_tblname_refresh}"` command refreshes the metadata for the aggregated table, making the new data visible to downstream reporting jobs. |
| **SDP ticketing system** | Failure notifications are routed to `insdp@tatacommunications.com` via `mailx`; the ticketing system consumes these emails. |
| **MOVE development team** | Receives failure emails (via `$MailId` and `$MOVE_DEV_TEAM`). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent executions** (process‑count guard may miss fast‑starting duplicates) | Duplicate Java jobs → data corruption or resource contention | Replace the `ps | grep` guard with a file‑based lock (e.g., `flock`) or a dedicated lock directory. |
| **Stale status flag** (script crashes before resetting flag) | Subsequent runs abort incorrectly | Implement a watchdog that clears flags after a configurable timeout; add a “recover” mode that forces reset after manual verification. |
| **Java job failure not captured** (non‑zero exit but script continues) | Inconsistent data in Impala | Ensure the Java class returns proper exit codes; add explicit error‑log parsing if needed. |
| **Impala refresh failure** (network or syntax error) | Downstream reports see stale data | Capture the exit status of `impala-shell`; on failure, trigger the same failure path (email/SDP). |
| **Email/SDP delivery failure** (mailx mis‑configured) | No alert reaches ops team | Verify mail service health; add a fallback log entry and optionally a secondary alert channel (e.g., Slack webhook). |
| **Missing/incorrect properties** (variables undefined) | Script aborts early, no logs | Add a validation block after sourcing the properties file that checks for required variables and exits with a clear message if any are missing. |

---

## 6. Running / Debugging the Script  

1. **Standard execution (via scheduler)**  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Daily_SimInventory_tbl_Aggr.sh
   ```
   The script writes logs to `$MNAAS_DailySimInventoryAggrLogPath$(date +_%F)`.

2. **Manual run (for testing)**  
   ```bash
   export MNAASDailySimInventoryAggrScriptName=MNAAS_Daily_SimInventory_tbl_Aggr.sh   # ensure the guard matches
   ./MNAAS_Daily_SimInventory_tbl_Aggr.sh
   ```
   - Check the console output (set `-x` already enabled).  
   - Verify the status file updates and that a new log file appears.

3. **Debugging steps**  
   - **Validate properties**: `grep -E 'MNAAS_.*=' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`  
   - **Force a failure**: temporarily change the Java class name to a non‑existent value; confirm that the failure path logs, sends email, and creates an SDP ticket.  
   - **Inspect lock logic**: run `ps -ef | grep -w $MNAASDailySimInventoryAggrScriptName` while the script is active to ensure the guard works.  
   - **Check Impala**: after a successful run, run the refresh query manually via `impala-shell` to confirm it succeeds.  

4. **Log locations**  
   - Daily log: `<log_path>/<date>.log` (e.g., `/var/log/mnaas/siminventory/2025-12-04.log`).  
   - System logger entries can be viewed with `journalctl -u syslog | grep MNAAS_Daily_SimInventory` (if syslog is used).  

---

## 7. External Configurations & Environment Variables  

| Variable (from properties) | Purpose | Typical Value Example |
|----------------------------|---------|-----------------------|
| `MNAAS_Daily_SimInventory_Aggr_ProcessStatusFileName` | Path to the shared status file used for coordination. | `/app/hadoop_users/MNAAS/status/siminventory_aggr.status` |
| `MNAAS_DailySimInventoryAggrLogPath` | Directory where daily log files are written. | `/app/hadoop_users/MNAAS/logs/siminventory_aggr/` |
| `Dname_MNAAS_Insert_Daily_Aggr_SimInventory_tbl` | Identifier for the Java aggregation process (used in `ps` check). | `InsertDailyAggrSimInv` |
| `Insert_Daily_Aggr_table` | Fully‑qualified Java class name to execute. | `com.tata.mnaas.aggregation.InsertDailyAggr` |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. | `impala-prod.tata.com` |
| `aggr_siminventory_daily_tblname_refresh` | Impala SQL statement to refresh the aggregated table. | `REFRESH TABLE mnaas.siminventory_daily` |
| `MailId`, `ccList`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email routing information for failure notifications. | `move-dev@tatacommunications.com` etc. |
| `MNAAS_Property_filename`, `MNAAS_Daily_SimInventory_AggrCntrlFileName`, `processname_siminventory` | Arguments passed to the Java job (control file, process name, etc.). | See property file for concrete paths. |

---

## 8. Suggested Improvements (TODO)

1. **Replace ad‑hoc `ps | grep` locking with `flock`** – ensures atomic acquisition of a lock file and eliminates race conditions when two scheduler instances start almost simultaneously.  
2. **Add explicit validation of required properties** immediately after sourcing the properties file; exit early with a clear error message if any variable is missing or empty.  

---