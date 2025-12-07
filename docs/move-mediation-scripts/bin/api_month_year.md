**File:** `move-mediation-scripts\bin\api_month_year.sh`  

---

### 1. High‑Level Summary
`api_month_year.sh` orchestrates the nightly export of two aggregated count tables – **MONTH** and **YEAR** – from the Hadoop/Hive data lake to an Oracle reporting database. It truncates the target Oracle tables, loads the corresponding non‑partitioned Hive tables into HDFS, and then uses Sqoop to export the data to Oracle. The script maintains a shared *process‑status* file to coordinate execution, enforce a single‑instance lock, and record success/failure flags for downstream monitoring and alerting.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `move_curr_month_count_table_Sqoop` | * Truncates the Hive staging table for month counts (`move_curr_month_count_non_part`).<br>* Loads fresh data into the staging table.<br>* Truncates the Oracle target `MOVE_CURR_MONTH_COUNT` via `sqlplus`.<br>* Executes a Sqoop export from HDFS to Oracle.<br>* Updates the process‑status flag to **1** and logs success/failure. |
| `move_curr_year_count_table_Sqoop` | Same steps as above but for the year‑level table `MOVE_CURR_YEAR_COUNT`. Updates the status flag to **2**. |
| `terminateCron_successful_completion` | Resets status flags to *idle* (`MNAAS_Daily_ProcessStatusFlag=0`), records a successful run timestamp, and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, records run timestamp, and exits with code 1. (Optionally triggers email/SDP ticket via `email_and_SDP_ticket_triggering_step`). |
| `email_and_SDP_ticket_triggering_step` | Sends a failure notification email (and marks an SDP ticket as created) the first time a failure occurs for the current run. |
| **Main program** (bottom of file) | Enforces a *single‑process* lock using the PID stored in the status file, reads the current flag, decides which export(s) to run, and invokes the appropriate termination routine. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_api_med_data.properties` (sourced at start) – provides paths, DB credentials, Impala host, Hive table names, etc.<br>• Environment variables set in the properties file: `MNAASLocalLogPath`, `MNAASConfPath`, `IMPALAD_HOST`, `move_curr_month_count_non_part_tblname_truncate`, `move_curr_month_count_non_part_tblname_load`, `move_curr_year_count_non_part_tblname_truncate`, `move_curr_year_count_non_part_tblname_load`, `OrgDetails_*` (Oracle connection), `ccList`, `GTPMailId`, `MNAAS_Sqoop_api_med_data_Scriptname`, etc.<br>• HDFS data under `/user/hive/warehouse/mnaas.db/move_curr_month_count_non_part` and `/move_curr_year_count_non_part`. |
| **Outputs** | • Oracle tables `MOVE_CURR_MONTH_COUNT` and `MOVE_CURR_YEAR_COUNT` populated with the latest aggregates.<br>• Log file `$MNAASLocalLogPath/MNAAS_api_med_data.log_YYYY-MM-DD` (appended with timestamps).<br>• Updated status file `$MNAASConfPath/api_med_data_ProcessStatusFile` (flags, PID, timestamps, job status). |
| **Side Effects** | • Truncates Hive staging tables (via Impala).<br>• Truncates Oracle target tables (via `sqlplus`).<br>• Sends email on first failure (via `mail`). |
| **Assumptions** | • Oracle client (`$ORACLE_HOME`) is installed and reachable.<br>• Impala service is reachable at `$IMPALAD_HOST`.<br>• The status file exists and is writable by the script user.<br>• The Oracle credentials (`move/move123`) are valid (hard‑coded in script).<br>• HDFS paths are correct and contain the expected CSV‑style data (`;` delimiter, `\N` nulls). |

---

### 4. Integration Points & Connections  

| Component | How `api_month_year.sh` Interacts |
|-----------|-----------------------------------|
| **`MNAAS_api_med_data.properties`** | Central configuration source for all paths, DB connection strings, table names, and email settings. |
| **Impala** | Used via `impala-shell` to truncate and load Hive staging tables (`*_tblname_truncate` / `*_tblname_load`). |
| **Oracle** | `sqlplus` runs a separate script (`truncate_table_move_curr_month_count.sh` / `truncate_table_move_curr_year_count.sh`) to empty target tables; Sqoop exports data to the same tables. |
| **Sqoop** | Performs the actual data movement from HDFS to Oracle. |
| **Process‑status file** (`api_med_data_ProcessStatusFile`) | Shared with other “MNAAS” scripts (e.g., `Move_Invoice_Register.sh`, `MNAAS_table_statistics_customer_aggr.sh`) to coordinate daily batch runs and to expose job status to monitoring dashboards. |
| **Mail system** | Sends failure alerts; the same mail logic appears in other scripts (e.g., `Move_Invoice_Register.sh`). |
| **Cron / Scheduler** | Typically invoked by a cron entry (not shown) that runs the script once per day. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Hard‑coded Oracle credentials** (`move/move123`) – security exposure. | Move credentials to a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and source them at runtime. |
| **Log file variable defined after redirection** (`exec 2>>$api_med_data_table_SqoopLogName`). | Define `api_med_data_table_SqoopLogName` *before* the `exec` line, or use `exec 2>>"$MNAASLocalLogPath/$(basename $0).log$(date +_%F)"`. |
| **No error handling for Impala commands** – failures may go unnoticed. | Capture exit codes (`$?`) after each `impala-shell` call and abort with `terminateCron_Unsuccessful_completion` if non‑zero. |
| **Single‑process lock based on stale PID** – if script crashes, PID may remain in status file. | Add a sanity check: verify the PID still belongs to this script (`ps -p $PID -o cmd=`) before assuming it’s running; clear stale PID on startup. |
| **Sqoop export runs with `-m 1`** (single mapper) – may be a performance bottleneck for large datasets. | Evaluate data volume; increase mapper count (`-m <n>`) after testing. |
| **Email/SDP ticket may be sent repeatedly** if status file isn’t reset. | Ensure `MNAAS_email_sdp_created` flag is cleared on successful runs (already done in `terminateCron_successful_completion`). |
| **Dependency on external scripts** (`truncate_table_move_curr_month_count.sh`, etc.) – if they change, this script may break. | Version‑control those scripts and validate their interfaces (expected parameters, exit codes). |

---

### 6. Running & Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Standard execution** (usually via cron) | `bash /app/hadoop_users/MNAAS/MNAAS_CronFiles/api_month_year.sh` |
| **Manual run with debug output** | `bash -x /app/hadoop_users/MNAAS/MNAAS_CronFiles/api_month_year.sh` (the script already contains `set -x`). |
| **Check logs** | `tail -f $MNAASLocalLogPath/MNAAS_api_med_data.log_$(date +%F)` |
| **Inspect status file** | `cat $MNAASConfPath/api_med_data_ProcessStatusFile` – look for flags, PID, job status. |
| **Force re‑run** (clear flag) | Edit the status file to set `MNAAS_Daily_ProcessStatusFlag=0` and remove the PID line, then invoke the script. |
| **Validate Impala commands** | Run the queries manually: `impala-shell -i $IMPALAD_HOST -q "<SQL>"` and verify success. |
| **Validate Sqoop export** | Run the Sqoop command with `--verbose` and check the exit code. |
| **Simulate failure** | Introduce a syntax error in one of the Impala queries, then run the script to confirm that failure handling (email, status flag) works. |

---

### 7. External Configuration & Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAASLocalLogPath` | Directory for log files. |
| `MNAASConfPath` | Directory containing the shared process‑status file. |
| `IMPALAD_HOST` | Hostname of the Impala daemon. |
| `move_curr_month_count_non_part_tblname_truncate` / `move_curr_month_count_non_part_tblname_load` | Impala SQL statements used to truncate/load the month‑count staging table. |
| `move_curr_year_count_non_part_tblname_truncate` / `move_curr_year_count_non_part_tblname_load` | Same for the year‑count table. |
| `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password` | Oracle JDBC connection details for Sqoop. |
| `ccList`, `GTPMailId` | Email recipients for failure notifications. |
| `MNAAS_Sqoop_api_med_data_Scriptname` | Human‑readable script identifier used in logs and emails. |
| `MNAAS_Script_Process_Id`, `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, etc. | Keys stored in the status file to coordinate runs. |

---

### 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace the hard‑coded Oracle user/password with a call to a secrets manager and inject them at runtime.  
2. **Robust Error Handling** – Add explicit exit‑code checks after each `impala-shell` and `sqlplus` invocation; on failure, invoke `email_and_SDP_ticket_triggering_step` before exiting.  

--- 

*End of documentation.*