**MNAAS_PPU_Report.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`MNAAS_PPU_Report.sh` is the daily orchestration driver for the “PPU” (Per‑Product‑Usage) reporting pipeline. It pulls subscriber and usage data from an Oracle source into partition‑ed Hive tables via Sqoop (currently disabled), refreshes the corresponding Impala metadata, and then invokes the Java `RAReport.jar` to generate daily (and, on the 1st of each month, monthly) PPU summary reports. The script updates a shared status file, logs progress, and on failure creates an SDP ticket and sends an alert email. It is scheduled to run each weekday at 10:45 am.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `ppu_daily_report_subs_table_Sqoop` | - Marks process flag = 1 in the status file.<br>- Removes stale HDFS lookup files.<br>- Drops the current month partition from the Hive subscriber summary table.<br>- Executes a Sqoop import from Oracle using the query defined in the properties file.<br>- Loads the imported files into the Hive partition and refreshes Impala. |
| `ppu_daily_report_usage_table_Sqoop` | Same as above but for the usage summary table (process flag = 2). |
| `MNAAS_PPU_Report_Jar_Execution` | - Sets process flag = 3.<br>- Calls `RAReport.jar` to generate the **daily** PPU report (`dppu`).<br>- If the current day is the 1st, also generates the **monthly** PPU report (`mppu`).<br>- Logs success/failure and triggers termination routine. |
| `terminateCron_successful_completion` | Resets status flag to 0, records success metadata (run time, job status) in the status file, logs completion, and exits 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, updates run‑time in the status file, calls `email_and_SDP_ticket_triggering_step`, and exits 1. |
| `email_and_SDP_ticket_triggering_step` | Marks job status as *Failure*; if an SDP ticket has not yet been created, sends a formatted email to the configured distribution list and flips the `MNAAS_email_sdp_created` flag to avoid duplicate tickets. |
| **Main program** | - Checks for an already‑running instance via PID stored in the status file.<br>- Updates the PID if free.<br>- Reads `MNAAS_Daily_ProcessStatusFlag` to decide which step(s) to run (currently only the Java jar step is active).<br>- Calls the appropriate function(s) and finalises with a success or failure termination routine. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`. Expected variables include:<br>• `ppu_daily_report_ProcessStatusFileName` – path to the shared status file.<br>• `ppu_daily_report_Scriptname` – script identifier used in the status file.<br>• Hive DB/table names (`dbname`, `ppu_daily_report_subs_table_name`, `ppu_daily_report_usage_table_name`).<br>• HDFS directories (`*_Dir`).<br>• Oracle connection params (`ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`).<br>• Queries (`*_Query`).<br>• Log path (`MNAAS_RAReport_LogPath`).<br>• Impala host (`IMPALAD_HOST`).<br>• Email recipients (`GTPMailId`, `ccList`). |
| **External Services** | - Oracle database (source data).<br>- Hadoop HDFS (temporary import directory).<br>- Hive Metastore (table partitions).<br>- Impala (metadata refresh).<br>- Java `RAReport.jar` (report generation).<br>- SMTP server (mail command). |
| **Inputs (runtime)** | No command‑line arguments are used; the script relies entirely on the property file and the current system date. |
| **Outputs** | - Hive tables `gen_daily_subs_ppu_summary` and `gen_daily_usage_ppu_summary` (partitioned by month).<br>- Daily (and possibly monthly) PPU report files produced by `RAReport.jar` and placed in the Data Lake (path defined inside the jar).<br>- Log file at `$MNAAS_RAReport_LogPath`.<br>- Updated status file (`$ppu_daily_report_ProcessStatusFileName`). |
| **Side Effects** | - Deletes any existing files under the import directories before each Sqoop run.<br>- Sends an alert email and creates an SDP ticket on failure.<br>- Updates a PID entry to prevent concurrent runs. |
| **Assumptions** | - The property file is correct and accessible to the user running the script.<br>- Oracle credentials have sufficient SELECT rights for the supplied queries.<br>- Hive/Impala services are up and the target tables exist with the expected partition schema.<br>- The `mail` command is configured and can reach the SMTP server.<br>- The status file is writable and not corrupted. |

---

### 4. Interaction with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Other “MNAAS_*** scripts (e.g., `MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh`) | Share the same status‑file convention (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, etc.) to coordinate daily batch windows. |
| **`RAReport.jar`** | Consumed here to produce the PPU reports; the jar is also used by other reporting scripts (e.g., monthly RA reports). |
| **Sqoop import scripts** (`ppu_daily_report_subs_table_Sqoop`, `ppu_daily_report_usage_table_Sqoop`) | Similar functions exist in other scripts that load different data sets; they follow the same pattern of flag updates and Hive loading. |
| **SDP ticketing system** | Triggered via email; downstream monitoring expects the “SDP ticket created” flag to be set. |
| **Cron scheduler** | The script is invoked daily at 10:45 am via a crontab entry (not shown). |
| **HDFS / Hive / Impala** | Tables populated here are later consumed by downstream analytics pipelines (e.g., billing, usage dashboards). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale / corrupted status file** – script may think a previous run is still active. | Job skip or duplicate runs. | Add a health‑check that validates the PID is still alive; rotate the status file daily. |
| **Oracle connectivity failure** – Sqoop import (if re‑enabled) will fail. | Missing data in Hive tables. | Implement retry logic with exponential back‑off; alert on repeated failures. |
| **Hive/Impala metadata out‑of‑sync** – `refresh` may not succeed. | Queries on the tables return stale data. | Verify `impala-shell` exit code; fallback to `invalidate metadata`. |
| **Permission issues on HDFS directories** – `chmod 777` may be insufficient on secured clusters. | Import files cannot be read/written. | Use proper HDFS ACLs; log the exact `hadoop fs -chmod` result. |
| **Email delivery failure** – SDP ticket not created. | No visibility of failure for ops team. | Capture `mail` exit status; fallback to a secondary notification channel (e.g., Slack, PagerDuty). |
| **Concurrent execution** – two cron instances start before PID file is updated. | Data corruption or duplicate processing. | Use a lock file (`flock`) in addition to PID check. |
| **Hard‑coded paths** – script assumes absolute locations. | Breakage after directory restructure. | Externalize all paths into the property file and validate at start‑up. |

---

### 6. Running / Debugging the Script  

1. **Standard execution** (as scheduled):  
   ```bash
   /app/hadoop_users/MNAAS/bin/MNAAS_PPU_Report.sh
   ```
   The script writes detailed logs to `$MNAAS_RAReport_LogPath`.

2. **Manual run (debug mode)**:  
   ```bash
   set -x   # already enabled via `set -vx` in script
   ./MNAAS_PPU_Report.sh
   ```
   Observe console output and the log file for step‑by‑step messages.

3. **Force a specific step** (e.g., only the Sqoop load):  
   - Uncomment the relevant function calls in the main block (`ppu_daily_report_subs_table_Sqoop` / `ppu_daily_report_usage_table_Sqoop`).  
   - Reset the flag in the status file to `0` or `1` as appropriate:  
     ```bash
     sed -i 's/MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=0/' $ppu_daily_report_ProcessStatusFileName
     ```

4. **Check PID / lock status**:  
   ```bash
   grep MNAAS_Script_Process_Id $ppu_daily_report_ProcessStatusFileName
   ps -p <PID>
   ```

5. **Validate external dependencies**:  
   - Test Oracle connectivity: `sqoop eval --connect jdbc:oracle:thin:@//<host>:<port>/<service> --username $ora_usernameMOVE --password $ora_passwordMOVE -e "SELECT 1 FROM dual;"`  
   - Verify Hive table exists: `hive -e "SHOW TABLES LIKE '${ppu_daily_report_subs_table_name}';"`  
   - Test Impala refresh: `impala-shell -i $IMPALAD_HOST -q "refresh ${dbname}.${ppu_daily_report_subs_table_name};"`  

6. **Inspect email/SDP ticket**:  
   - Look for the email in the mailbox of `$GTPMailId`.  
   - Confirm the ticket creation flag in the status file (`MNAAS_email_sdp_created=Yes`).  

---

### 7. External Configurations & Environment Variables  

| Variable (from property file) | Description | Typical Usage |
|-------------------------------|-------------|---------------|
| `ppu_daily_report_ProcessStatusFileName` | Path to the shared status file that tracks flags, PID, timestamps, etc. | Updated throughout the script via `sed`. |
| `ppu_daily_report_Scriptname` | Logical name of this script (e.g., `MNAAS_PPU_Report.sh`). | Written to status file for PID tracking. |
| `dbname` | Hive database containing the PPU summary tables. | Used in Hive/Impala commands. |
| `ppu_daily_report_subs_table_name` / `ppu_daily_report_usage_table_name` | Hive table names for subscriber and usage summaries. | Target of Sqoop load and Impala refresh. |
| `ppu_daily_report_subs_table_Dir` / `ppu_daily_report_usage_table_Dir` | HDFS staging directories for Sqoop imports. | Cleaned before each import. |
| `ppu_daily_report_subs_table_Query` / `ppu_daily_report_usage_table_Query` | Oracle SQL queries (with `$CONDITIONS` placeholder) passed to Sqoop. | Determines data extracted. |
| `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE` | Oracle connection details. | Used by Sqoop. |
| `MNAAS_RAReport_LogPath` | Full path to the log file for this run. | All `logger` statements append here. |
| `IMPALAD_HOST` | Hostname of the Impala daemon for metadata refresh. | Used by `impala-shell`. |
| `GTPMailId`, `ccList` | Email recipients for failure notifications. | Used by `mail`. |
| `MNAAS_FlagValue` (derived) | Current process flag read from the status file. | Drives conditional execution flow. |

---

### 8. Suggested Improvements (TODO)

1. **Re‑enable & robustify the Sqoop steps**  
   - Remove the comment markers and add retry logic (e.g., `sqoop import … || { logger …; exit 1; }`).  
   - Parameterize the number of mappers (`-m`) based on data volume.

2. **Replace ad‑hoc PID lock with `flock`**  
   - Wrap the entire script body in `flock -n /tmp/MNAAS_PPU_Report.lock || { logger "Another instance running"; exit 0; }` to guarantee atomic lock acquisition and automatic cleanup.

3. *(Optional)* **Externalize email parameters** into the property file and verify `mail` exit status; fallback to a secondary alert channel if email fails.

--- 

*End of documentation.*