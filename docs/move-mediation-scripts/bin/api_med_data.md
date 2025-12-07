**File:** `move-mediation-scripts/bin/api_med_data.sh`

---

## 1. Purpose (one‑paragraph summary)

`api_med_data.sh` orchestrates the daily refresh of the **API‑MED** data set used by the MNAAS analytics platform. It truncates and repopulates several Impala tables from pre‑computed views, then exports the consolidated data to an Oracle data‑warehouse via Sqoop. The script also maintains a shared *process‑status* file that is read by downstream jobs, logs every step, and implements simple concurrency control to avoid overlapping runs.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **api_med_data_table_load** | Truncates and reloads Impala tables `api_med_1_7_30d`, `api_med_last_values`, and `api_med_data` from their corresponding views. Updates the status flag to **1** (load phase). |
| **api_med_data_table_Sqoop** | Sets status flag **2**, truncates the target Oracle table (`API_MED_DATA`) via a SQL*Plus script, then exports the HDFS Hive table `api_med_data` to Oracle using Sqoop. Handles success/failure logging. |
| **move_curr_month_count_table_Sqoop** | Sets status flag **3**, truncates & loads a non‑partitioned Impala table for current‑month counts, then Sqoop‑exports it to Oracle table `MOVE_CURR_MONTH_COUNT`. |
| **move_curr_year_count_table_Sqoop** | Sets status flag **4**, similar to the month‑count step but for yearly aggregates (`MOVE_CURR_YEAR_COUNT`). |
| **terminateCron_successful_completion** | Resets the status file to “idle” (`MNAAS_Daily_ProcessStatusFlag=0`), marks the job as **Success**, records run time, and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, updates run‑time, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | On first failure, sends an alert e‑mail to the IPX team and marks an SDP ticket as created (guarded by a flag to avoid duplicate alerts). |
| **Main program** | Performs a PID‑based concurrency check, reads the current status flag, and dispatches the appropriate phase(s). Most phases are currently commented out, leaving only the Sqoop export of `api_med_data` active. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_api_med_data.properties`. Expected variables include:<br>• `api_med_data_table_SqoopLogName` – log file path<br>• `api_med_data_Sqoop_ProcessStatusFile` – shared status file<br>• `IMPALAD_HOST` – Impala daemon host<br>• `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password` – Oracle JDBC connection details<br>• `dbname`, `api_med_data_table_name`, `move_curr_month_count`, `move_curr_year_count` – logical names used in logs<br>• `MNAAS_Sqoop_api_med_data_Scriptname`, `ccList`, `GTPMailId` – for e‑mail alerts |
| **External Services** | • **Impala** – executed via `impala-shell` to truncate/load tables.<br>• **Oracle** – accessed via `sqlplus` (local script) and Sqoop export.<br>• **Sqoop** – moves data from HDFS (`/user/hive/warehouse/mnaas.db/...`) to Oracle.<br>• **Mail** – `mail` command for failure notifications. |
| **Data Stores** | • **HDFS / Hive** – source tables `api_med_data`, `move_curr_month_count_non_part`, `move_curr_year_count_non_part`.<br>• **Impala** – target tables `mnaas.api_med_*`.<br>• **Oracle** – destination tables `API_MED_DATA`, `MOVE_CURR_MONTH_COUNT`, `MOVE_CURR_YEAR_COUNT`. |
| **Outputs** | • Log file (`$api_med_data_table_SqoopLogName`).<br>• Updated status file (`$api_med_data_Sqoop_ProcessStatusFile`).<br>• Populated Oracle tables (visible to downstream reporting). |
| **Side Effects** | • Overwrites (truncates) target tables each run.<br>• Alters the shared status file, which other scripts read to decide whether to start. |
| **Assumptions** | • The properties file exists and defines all required variables.<br>• Impala, Hive, and Oracle services are reachable from the host.<br>• The Oracle user/password (`move/move123`) is valid and has INSERT privileges.<br>• Only one instance runs at a time (PID check).<br>• HDFS paths are correctly populated by upstream ETL jobs. |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Status File (`$api_med_data_Sqoop_ProcessStatusFile`)** | Shared with many other MNAAS scripts (e.g., `MNAAS_Usage_Trend_Aggr.sh`, `MNAAS_table_statistics_customer_aggr.sh`). Those scripts read the flag to determine if the API‑MED data is ready. |
| **Truncate Scripts (`truncate_table_*.sh`)** | Called via `sqlplus` to clean Oracle tables before Sqoop export. These scripts reside in `/app/hadoop_users/MNAAS/MNAAS_CronFiles/`. |
| **Downstream Reporting** | Oracle tables populated here are consumed by BI tools (Tableau, custom dashboards) and possibly by other batch jobs that generate invoices or usage aggregates. |
| **Cron Scheduler** | Typically invoked from a daily cron entry (e.g., `0 2 * * * /path/api_med_data.sh`). The same cron may also schedule related scripts that depend on the flag becoming `0`. |
| **Mail / SDP System** | Failure alerts are sent to the IPX team; the script sets a flag to avoid duplicate tickets. Other monitoring tools may watch the log file for the “failed” string. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded Oracle credentials** (`move/move123`) | Credential leakage, unauthorized data changes. | Move credentials to a secure vault (e.g., Hadoop Credential Provider) and reference via `--password-file`. |
| **Single‑threaded Sqoop (`-m 1`)** | Long run‑time, possible bottleneck on large data volumes. | Increase parallelism (`-m <num>`) after testing; ensure Oracle can handle concurrent inserts. |
| **No error handling for Impala commands** | Silent failures could leave tables empty while status flag shows success. | Capture exit codes of each `impala-shell` call; abort and set failure flag on non‑zero status. |
| **PID check race condition** | Two instances could start if the status file is edited between check and write. | Use a lock file (`flock`) or atomic `mkdir` lock pattern. |
| **Sed in‑place edits on shared status file** | Concurrent edits may corrupt the file. | Serialize access via the same lock mechanism used for PID check. |
| **Mail command may block** | If the mail subsystem hangs, the script could stall. | Run mail in background or use a non‑blocking mail API; add timeout. |
| **Truncate‑then‑load pattern** | If the load fails, data is lost for the day. | Consider using a staging table and swap (rename) after successful load. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file exists and is readable.  
   - Verify that `impala-shell`, `sqoop`, `sqlplus`, and `mail` are in the `$PATH`.  
   - Confirm HDFS directories (`/user/hive/warehouse/mnaas.db/...`) contain the latest data.  

2. **Execution**  
   ```bash
   chmod +x api_med_data.sh
   ./api_med_data.sh
   ```
   The script writes its PID to the status file and logs to `$api_med_data_table_SqoopLogName`.

3. **Debug Mode**  
   - The script already contains `set -x` (debug trace).  
   - To increase verbosity, export `HADOOP_ROOT_LOGGER=DEBUG,console` before running.  

4. **Checking Results**  
   - Review the log file for lines ending with “loaded”, “finished”, or “failed”.  
   - Query the status file to see the final flag (`grep MNAAS_Daily_ProcessStatusFlag`).  
   - Verify Oracle tables via SQL*Plus: `SELECT COUNT(*) FROM API_MED_DATA;`.  

5. **Manual Step‑by‑Step**  
   - Run individual functions in a subshell for isolation, e.g.:  
     ```bash
     source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_api_med_data.properties
     api_med_data_table_load
     api_med_data_table_Sqoop
     ```
   - Capture exit codes after each call (`echo $?`).  

---

## 7. External Configurations & Files Referenced

| File | Purpose |
|------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_api_med_data.properties` | Defines all environment variables used throughout the script (paths, DB credentials, hostnames, log names). |
| `/app/hadoop_users/MNAAS/MNAAS_CronFiles/truncate_table_api_med_data.sh` | SQL*Plus script that truncates the Oracle `API_MED_DATA` table before export. |
| `/app/hadoop_users/MNAAS/MNAAS_CronFiles/truncate_table_move_curr_month_count.sh` | Truncates Oracle `MOVE_CURR_MONTH_COUNT`. |
| `/app/hadoop_users/MNAAS/MNAAS_CronFiles/truncate_table_move_curr_year_count.sh` | Truncates Oracle `MOVE_CURR_YEAR_COUNT`. |
| `$api_med_data_Sqoop_ProcessStatusFile` (path defined in properties) | Shared status/lock file used for concurrency control and inter‑script coordination. |
| `$api_med_data_table_SqoopLogName` (path defined in properties) | Central log file for this script’s execution. |

---

## 8. Suggested Improvements (TODO)

1. **Add robust error handling for Impala operations** – capture each `impala-shell` exit status, abort on failure, and set the status flag to a distinct error code (e.g., `5`).  
2. **Externalize sensitive credentials** – replace the hard‑coded Oracle user/password with a reference to a Hadoop Credential Provider or a secure vault, and modify the Sqoop command to use `--password-file`.  

--- 

*End of documentation.*