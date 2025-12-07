**Move_Sim_Inventory_Aggr.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`Move_Sim_Inventory_Aggr.sh` is the orchestration driver for the **monthly Sim‑Inventory aggregation** process in the MNAAS mediation platform. It coordinates the truncation of the target Hive aggregation table, launches a Java job that populates the table with aggregated SIM‑inventory data, refreshes the corresponding Impala metadata, and maintains a shared status file that records process state, timestamps, and success/failure flags. On failure it automatically sends an email notification and raises an SDP ticket. The script is guarded against concurrent executions and logs all activity to a daily log file.

---

### 2. Key Functions & Responsibilities

| Function / Block | Responsibility |
|------------------|----------------|
| **`MNAAS_insert_into_aggr_table`** | *Pre‑process*: sets “in‑progress” flags in the status file, logs start, ensures no duplicate Java job is running. <br>*Core*: truncates Hive aggregation table, invokes the Java class (`$Insert_Aggr_table`) with required properties, checks exit code. <br>*Post‑process*: on success runs an Impala `REFRESH` statement, logs success; on failure logs error and triggers termination. |
| **`terminateCron_successful_completion`** | Clears “in‑progress” flag, marks job as `Success`, records run time, writes final log entries, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, calls `email_and_SDP_ticket_triggering_step`, exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates status file to `Failure`, checks whether an alert has already been sent, sends a formatted email to the development team and an SDP ticket via `mailx`, updates the “email sent” flag, and logs the alert actions. |
| **Main script block** | Guard against concurrent script instances, read the daily process flag from the shared status file, decide whether to run the aggregation (flags 0/1) or abort, and invoke the above functions accordingly. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_ShellScript.properties` – defines all `$MNAAS_*` variables, classpath, jar locations, table names, email lists, IMPALA host, etc. |
| **External files** | • `$MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName` – plain‑text status file (key/value). <br>• `$sim_inventory_status_Aggr_Log$(date +_%F)` – daily log file (appended). |
| **Databases / Services** | • **Hive** – truncates `$dbname.$Move_siminventory_status_aggr_tblname`. <br>• **Impala** – runs `$Move_siminventory_status_aggr_tblname_refresh` via `impala-shell`. <br>• **Java job** – `$Insert_Aggr_table` class (in `$MNAAS_Main_JarPath`). |
| **Notifications** | • Email via `mail` (to `$GTPMailId` and `$ccList`). <br>• SDP ticket via `mailx` to `insdp@tatacommunications.com`. |
| **Outputs** | • Populated Hive/Impala aggregation table. <br>• Updated status file (flags, timestamps, job status). <br>• Log file entries. <br>• Optional email/SDP ticket on failure. |
| **Assumptions** | • All variables referenced in the properties file are defined and point to reachable resources. <br>• Hive, Impala, and the Java runtime are operational and accessible from the host. <br>• Mail utilities (`mail`, `mailx`) are configured and can send external mail. <br>• The status file is writable by the script user and not concurrently edited by other processes. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **Other MNAAS move scripts** (e.g., `MNAAS_move_files_from_staging_*`, `MNAAS_report_data_loading.sh`) | Produce source data files that the Java aggregation job consumes; they may also update the same status file for earlier steps. |
| **Java aggregation class** (`$Insert_Aggr_table`) | Called from this script; implements the actual data transformation and insertion logic. |
| **Status‑file based coordination** | Many MNAAS scripts read/write `$MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName` to serialize daily/monthly jobs. |
| **Monitoring / Scheduler** | Typically invoked by a cron entry (e.g., `0 2 1 * * /path/Move_Sim_Inventory_Aggr.sh`). The script’s own PID checks prevent overlapping runs. |
| **Support ticketing system** | Uses `mailx` to raise an SDP ticket; downstream processes (e.g., incident management) rely on this. |
| **Logging infrastructure** | Log files are consumed by log aggregation tools (e.g., Splunk, ELK) for operational visibility. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (multiple script instances) | Duplicate truncation / data loss | The script already checks `ps aux`; reinforce with a lock file (`flock`) to guarantee atomicity. |
| **Stale or corrupted status file** | Wrong flag values → job skipped or runaway | Add validation of required keys and fallback defaults; archive old status files daily. |
| **Java job failure (non‑zero exit)** | Aggregation table remains empty, downstream reports broken | Capture Java stdout/stderr to a separate log, implement retry logic with exponential back‑off. |
| **Hive/Impala connectivity loss** | Truncate or refresh fails, script hangs | Set reasonable timeouts on Hive/Impala commands; on timeout, trigger failure path and alert. |
| **Email/SDP ticket delivery failure** | No alert sent, issue goes unnoticed | Verify mailx return code; if non‑zero, write to a “fallback” alert file and raise a system alarm. |
| **Hard‑coded date formats in log filenames** | Log rotation may miss files across month boundaries | Use a consistent naming scheme (e.g., `$(date +%Y%m%d)`) and ensure log rotation policies handle them. |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the user has read/write access to the properties file, status file, and log directory.  
   - Verify that Hive, Impala, Java, `mail`, and `mailx` are in the `$PATH`.  
   - Confirm that the environment variable `CLASSPATHVAR` and `$MNAAS_Main_JarPath` point to valid locations.

2. **Manual execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/
   ./move-mediation-scripts/bin/Move_Sim_Inventory_Aggr.sh
   ```
   - The script prints each command (`set -x`) and writes detailed logs to  
     `$sim_inventory_status_Aggr_Log$(date +_%F)`.

3. **Debugging steps**  
   - **Check the status file**: `cat $MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName` – verify `MNAAS_Daily_ProcessStatusFlag` is 0 or 1.  
   - **Inspect the log**: `tail -f $sim_inventory_status_Aggr_Log$(date +_%F)`.  
   - **Validate Java job**: Run the Java class manually with the same arguments to see stdout/stderr.  
   - **Force failure**: Temporarily change the flag to an unexpected value to test the failure path and email/SDP ticket generation.  
   - **PID check**: `ps -ef | grep Move_Sim_Inventory_Aggr.sh` – ensure no stray processes remain after completion.

4. **Cron integration**  
   - Typical crontab entry (run on the first day of each month at 02:00):  
     ```cron
     0 2 1 * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/Move_Sim_Inventory_Aggr.sh >> /var/log/mnaas/Move_Sim_Inventory_Aggr_cron.log 2>&1
     ```

---

### 7. External Configuration & Environment Variables

| Variable (populated in `MNAAS_ShellScript.properties`) | Role |
|------------------------------------------------------|------|
| `MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName` | Path to the shared status file. |
| `sim_inventory_status_Aggr_Log` | Base name for daily log files. |
| `dbname` | Hive database name. |
| `Move_siminventory_status_aggr_tblname` | Hive aggregation table name. |
| `Dname_Move_siminventory_status_aggr_tblname` | Identifier used for the Java process lock check. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath` | Java runtime classpath and jar location. |
| `Insert_Aggr_table` | Fully‑qualified Java class name that performs the aggregation. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. |
| `Move_siminventory_status_aggr_tblname_refresh` | Impala `REFRESH` statement (e.g., `REFRESH $dbname.$Move_siminventory_status_aggr_tblname`). |
| `MNAASSimInventoryStatusAggrScriptName` | Script name used in logs and alerts. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / sender details for alerts. |
| `MNAAS_email_sdp_created` flag | Tracks whether an alert has already been sent for the current run. |

If any of these variables are missing or point to non‑existent resources, the script will fail early (e.g., `sed` will error, Java will not start, mail will bounce). Validation of the properties file at script start is recommended.

---

### 8. Suggested Improvements (TODO)

1. **Lock‑file implementation** – Replace the ad‑hoc `ps` checks with a robust `flock`‑based lock file to guarantee single‑instance execution even across node reboots or user‑initiated kills.  
2. **Centralised error handling** – Extract repeated `logger`/`sed` patterns into a small helper library (e.g., `MNAAS_utils.sh`) to reduce duplication and make future status‑field changes easier.

---