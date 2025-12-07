**MNAAS_SIM_IMSI_ageing_report_load.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`MNAAS_SIM_IMSI_ageing_report_load.sh` is a daily batch driver that populates three Hive tables used for the SIM‑IMSI ageing report: a temporary SIM table, a temporary traffic table, and the final report table. It orchestrates the load steps, updates a shared process‑status file, writes detailed logs, and on failure raises an SDP ticket via email. The script is guarded against concurrent execution and can resume from an intermediate step based on a persisted flag.

---

### 2. Key Functions / Logical Units  

| Function | Responsibility |
|----------|----------------|
| **load_temp_sim_table** | Truncates and reloads the temporary SIM ageing Hive table using the query defined in the properties file. Updates status flag = 1. |
| **load_temp_traffic_table** | Truncates and reloads the temporary traffic ageing Hive table using the query from the properties file. Updates status flag = 2. |
| **load_report_table** | Truncates and reloads the final report Hive table using the query from the properties file. Updates status flag = 3 and terminates with success. |
| **terminateCron_successful_completion** | Writes “Success” into the process‑status file, logs completion timestamps, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers an email‑based SDP ticket, and exits with code 1. |
| **email_on_reject_triggering_step** | Builds a minimal RFC‑822 body and sends it via `mailx` to the SDP mailbox, using sender/CC addresses supplied by environment variables. |
| **Main program** | Checks for an already‑running instance (PID guard), reads the persisted flag, decides which load steps to execute (allowing restart from flag 1‑3), and finally calls the appropriate termination routine. |

---

### 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Configuration / External Files** | `MNAAS_SIM_IMSI_ageing_report_load.properties` (defines table names, Hive queries, log path, status‑file path, script name, etc.). |
| **Environment Variables** | `setparameter` (evaluated at start), `$SDP_ticket_from_email`, `$SDP_ticket_cc_email` (used for failure email). |
| **Process‑Status File** | Path stored in `$MNAAS_SIM_IMSI_ageing_report_load_ProcessStatusFileName`. Holds keys: `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_Daily_ProcessName`, `MNAAS_job_status`, `MNAAS_Script_Process_Id`, etc. Updated throughout execution. |
| **Hive** | Executes three `truncate` + `INSERT` statements via the Hive CLI (`hive -e`). Requires Hive server access and proper Kerberos/impersonation if enabled. |
| **Logging** | Writes to `$MNAAS_SIM_IMSI_ageing_report_load_logpath` via `logger -s`. |
| **Email / SDP Ticket** | On failure, sends a mail to `insdp@tatacommunications.com` with a predefined template. |
| **Process Guard** | Uses `ps` to ensure only one instance runs; writes its PID into the status file. |
| **Exit Codes** | `0` on success, `1` on any failure (including concurrent‑run detection). |

**Assumptions**  
- The properties file exists and contains all required variables.  
- Hive CLI is in the PATH and the user has permission to truncate/insert into the target tables.  
- `logger`, `sed`, `mailx`, and standard Unix utilities are available.  
- The status file is writable by the script’s user.  
- SMTP configuration for `mailx` is functional.  

---

### 4. Interaction with Other Scripts / Components  

| Interaction | Description |
|-------------|-------------|
| **Shared Process‑Status File** | Many MNAAS daily scripts (e.g., `MNAAS_RawFileCount_Checks.sh`, `MNAAS_PreValidation_Checks.sh`) use the same status file pattern (`MNAAS_*_ProcessStatusFileName`). This file coordinates sequencing and prevents overlapping runs. |
| **Up‑stream Data Sources** | Raw SIM/traffic data is produced by ingestion pipelines (e.g., `MNAAS_RawFileCount_Checks.sh`, `MNAAS_Monthly_traffic_nodups_summation.sh`). Those pipelines must complete successfully before this script’s flag is reset to `0`. |
| **Down‑stream Consumers** | The final report table (`$SIM_IMSI_report_table`) is read by reporting jobs such as `MNAAS_PPU_Report.sh` or external BI tools. Those jobs typically start after this script finishes with `MNAAS_job_status=Success`. |
| **SDP Ticketing System** | Failure email triggers an incident in the SDP (Service Desk Platform) used by Tata Communications. Other scripts also use the same email template, ensuring a unified incident handling process. |
| **Cron Scheduler** | The script is intended to be invoked by a daily cron entry (e.g., `0 2 * * * /path/MNAAS_SIM_IMSI_ageing_report_load.sh`). The PID guard prevents overlapping cron runs. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Hive query failure (syntax, missing tables, permission)** | Validate queries in a dev environment; add `set -e` after each `hive -e` call or capture Hive error output and abort early. |
| **Stale PID / zombie process causing false “already running”** | Include a sanity check that the PID belongs to the expected script (`ps -fp $PID | grep $SCRIPT_NAME`). If not, clear the PID entry. |
| **Properties file corruption or missing variables** | Add a pre‑run validation block that `source` the file and `[[ -z $VAR ]] && exit 1` for each required variable. |
| **Unbounded log file growth** | Rotate logs via logrotate or implement size‑based truncation within the script. |
| **Email delivery failure masking a real failure** | Log the exit status of `mailx`; if non‑zero, write a secondary alert to a monitoring system (e.g., write to `/var/log/alerts`). |
| **Concurrent execution on multiple nodes** | Ensure the status file resides on a shared filesystem with atomic `sed` updates, or replace the file‑based lock with a distributed lock service (e.g., Zookeeper). |

---

### 6. Running / Debugging the Script  

1. **Manual Execution**  
   ```bash
   export setparameter="MNAAS_ENV=prod"
   /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SIM_IMSI_ageing_report_load.properties   # verify content
   ./MNAAS_SIM_IMSI_ageing_report_load.sh
   ```
   The script already runs with `set -x` (trace) and logs to the path defined in the properties file.

2. **Check Status**  
   ```bash
   cat $MNAAS_SIM_IMSI_ageing_report_load_ProcessStatusFileName
   tail -f $MNAAS_SIM_IMSI_ageing_report_load_logpath
   ```

3. **Force a Specific Step**  
   Edit the status file and set `MNAAS_Daily_ProcessStatusFlag=2` to start from the traffic‑table load only.

4. **Simulate Failure**  
   Introduce a syntax error in one of the Hive queries, re‑run, and verify that:  
   - The failure is logged.  
   - An email is sent.  
   - Exit code is `1`.  

5. **Verify No Duplicate Runs**  
   Start the script in one terminal, then quickly start it again in another; the second run should log “Previous … Process is running already” and exit with code 1.

---

### 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| `MNAAS_SIM_IMSI_ageing_report_load.properties` | Provides: `<table>_name`, `<table>_load_query`, `logpath`, `ProcessStatusFileName`, `Script` name, etc. |
| `setparameter` (evaluated via `eval $setparameter`) | Allows injection of runtime parameters (e.g., environment, date overrides). |
| `$SDP_ticket_from_email`, `$SDP_ticket_cc_email` | Sender and CC addresses for the failure notification email. |
| `mailx` | Sends the SDP ticket; requires proper `/etc/mail.rc` or equivalent configuration. |
| `hive` CLI | Executes the truncation and load queries; must be accessible to the script’s user. |
| `logger` | Writes to syslog and the script‑specific log file. |

---

### 8. Suggested Improvements (TODO)

1. **Add Robust Parameter Validation** – Before any Hive call, verify that all required variables (`*_table`, `*_load_query`, log and status paths) are non‑empty; abort with a clear error message if not.
2. **Replace File‑Based PID Guard with `flock`** – Use `exec 200>"$MNAAS_SIM_IMSI_ageing_report_load_ProcessStatusFileName.lock"` and `flock -n 200 || exit 1` to guarantee atomic exclusive execution, especially in a clustered environment.  

---