**Sim_Inventory_Sqoop.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`Sim_Inventory_Sqoop.sh` orchestrates the nightly/monthly load of SIM inventory data from an Oracle source into the Hadoop ecosystem (HDFS → Hive/Impala) and subsequently pushes aggregated results back to Oracle. It coordinates a series of Java‑based loaders, a Sqoop import, Hive/Impala table refreshes, and an export script, while maintaining a process‑status file that enables resumable execution, concurrency protection, and automated failure notification via email/SDP ticket.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **MNAAS_SimLoader** | Executes the `SIMLoader` Java JAR (pre‑processing of raw SIM files). Updates status flag to **1**. |
| **MNAAS_load_into_temp_table** | Performs Sqoop import from Oracle to HDFS, cleans previous staging data, copies files to a backup location, loads them into the Hive *intermediate* table, and refreshes Impala. Updates status flag to **2**. |
| **MNAAS_insert_into_main_table** | Calls a Java class (`Insert_Main_table`) to move data from the intermediate Hive table into the *main* SIM inventory table. Updates status flag to **3**. |
| **MNAAS_insert_into_aggr_table** | Calls a Java class (`Insert_Aggr_table`) to populate the *aggregated* SIM inventory table. Updates status flag to **4**. |
| **MNAAS_insert_into_sim_table** | Executes the `SIMTableLoader` Java JAR to load data into a final SIM table (often a “sim” reference table). Updates status flag to **5**. |
| **MNAAS_sqoop_export_into_oracle_table** | Triggers an external shell script (`api_month_year_ScriptName`) that exports processed data back to Oracle via Sqoop export. Updates status flag to **6**. |
| **terminateCron_successful_completion** | Resets status flags to **0**, marks job as *Success*, records run time, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Sends an email (via `mailx`) to the support team and creates an SDP ticket on first failure; updates status file to indicate ticket sent. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`. Expected variables include: <br>• `$sim_inventory_SqoopLog` (log file) <br>• `$MNAAS_SimInventory_Sqoop_ProcessStatusFileName` (status file) <br>• `$SIMLoaderJarPath`, `$SIMLoaderConfFilePath`, `$SIMTableLoaderJarPath`, `$MNAAS_Main_JarPath` <br>• Hive/Impala table names (`$dbname`, `$Move_siminventory_status_inter_tblname`, `$Move_siminventory_status_tblname`, `$Move_siminventory_status_aggr_tblname`) <br>• Oracle connection params (`$ora_serverNameMOVE`, `$ora_portNumberMOVE`, `$ora_serviceNameMOVE`, `$ora_usernameMOVE`, `$ora_passwordMOVE`) <br>• HDFS directories (`$HDFSSqoopDir`, `$Sim_inventory_backup`) <br>• Email/SDP vars (`$SDP_ticket_from_email`, `$MOVE_DEV_TEAM`, `$SDP_Receipient_List`) <br>• `$api_month_year_ScriptName` (export script) |
| **External Services** | Oracle DB (source & target), Hadoop HDFS, Hive, Impala, mailx (SMTP), SDP ticketing system. |
| **Primary Output** | Populated Hive/Impala tables (`*_status_inter_tblname`, `*_status_tblname`, `*_status_aggr_tblname`) and refreshed data in Oracle via export script. |
| **Logs** | All steps append to `$sim_inventory_SqoopLog`. Errors also go to the same log via `logger -s`. |
| **Process‑status file** | Updated throughout execution (flags 0‑6, process name, PID, job status, timestamps, email‑sent flag). |
| **Side Effects** | Deletion of previous staging files in HDFS, creation of backup copies, possible email/SDP ticket generation on failure, Impala metadata refreshes. |

---

### 4. Interaction with Other Scripts / Components  

| Component | Relationship |
|-----------|--------------|
| **MNAAS_ShellScript.properties** | Central configuration repository; many other “Move” scripts source the same file, ensuring consistent DB/HDFS paths and credentials. |
| **SIMLoader JAR** | Shared Java loader used by other SIM‑related ingestion scripts. |
| **Insert_Main_table / Insert_Aggr_table Java classes** | Common data‑movement utilities invoked by multiple “Move” pipelines (e.g., other inventory loads). |
| **api_month_year_ScriptName** | Separate shell script (not shown) that performs the Sqoop **export** back to Oracle; likely used by other monthly aggregation jobs. |
| **Process‑status file** | Same file (`$MNAAS_SimInventory_Sqoop_ProcessStatusFileName`) is read/written by other scripts that may need to resume or skip steps based on the flag value. |
| **Logging & Alerting** | The email/SDP logic mirrors that in other Move scripts (e.g., `Move_Invoice_Register.sh`), providing a unified failure‑notification mechanism. |
| **Cron Scheduler** | Typically invoked from a daily/weekly cron entry; other Move scripts follow the same pattern, allowing coordinated batch windows. |

---

### 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Missing/incorrect property values** (e.g., wrong Oracle password) | Validate required variables at script start; add a sanity‑check function that aborts with clear log messages if any are empty. |
| **Concurrent executions** (multiple instances could corrupt status file) | The script already checks PID from the status file; ensure the status file is placed on a reliable shared filesystem and that the PID check is atomic (e.g., use `flock`). |
| **Sqoop import failure** (network, schema change) | Capture Sqoop exit code, log the full command, and optionally retry a configurable number of times before aborting. |
| **Hive/Impala refresh failures** (metadata out‑of‑sync) | Verify the refresh command succeeded (`impala-shell` exit status) and add a fallback `invalidate metadata` if needed. |
| **Hard‑coded table/column names** may diverge from source schema | Keep schema definitions in a version‑controlled metadata file and reference them via variables rather than literals. |
| **Email/SDP spam on repeated failures** | The status file flag `MNAAS_email_sdp_created` prevents duplicate tickets; ensure it is reset on successful runs. |
| **Log file growth** (unbounded size) | Rotate `$sim_inventory_SqoopLog` via logrotate or implement size‑based truncation within the script. |

---

### 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` is present and populated.  
   - Verify Java, Sqoop, Hive, Impala, and `mailx` are in the `$PATH`.  
   - Confirm HDFS directories (`$HDFSSqoopDir`, `$Sim_inventory_backup`) exist and are writable.  

2. **Manual Execution**  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./Sim_Inventory_Sqoop.sh   # run as the same user that the cron job uses
   ```
   - The script will log to the file defined by `$sim_inventory_SqoopLog`.  
   - Check the process‑status file (`$MNAAS_SimInventory_Sqoop_ProcessStatusFileName`) for the final flag (`0` = success).  

3. **Debug Mode**  
   - The script starts with `set -x`, which prints each command to the log.  
   - To get more verbose output, tail the log while the job runs:  
     ```bash
     tail -f $sim_inventory_SqoopLog
     ```  

4. **Common Checks**  
   - **PID conflict**: `grep MNAAS_Script_Process_Id $MNAAS_SimInventory_Sqoop_ProcessStatusFileName` → ensure the PID is not running.  
   - **Sqoop command**: copy the generated command from the log and run it manually to isolate DB connectivity issues.  
   - **Java JARs**: run `java -jar $SIMLoaderJarPath` with `-h` (if supported) to verify the JAR is accessible.  

5. **Post‑run Validation**  
   - Verify Hive tables contain expected rows (`hive -e "select count(*) from $dbname.$Move_siminventory_status_tblname"`).  
   - Confirm Impala sees the data (`impala-shell -i $IMPALAD_HOST -q "select count(*) from $dbname.$Move_siminventory_status_tblname"`).  
   - Check Oracle target tables (via SQL*Plus or another client) if the export script succeeded.  

---

### 7. External Config / Files Referenced  

| File / Variable | Role |
|-----------------|------|
| **MNAAS_ShellScript.properties** | Central configuration (paths, DB credentials, table names, email settings). |
| **$MNAAS_SimInventory_Sqoop_ProcessStatusFileName** | Persistent status file that stores flags, PID, timestamps, and email‑sent flag. |
| **$SIMLoaderJarPath**, **$SIMTableLoaderJarPath**, **$MNAAS_Main_JarPath** | Java binaries that perform data transformations and loads. |
| **$SIMLoaderConfFilePath** | Configuration file read by the Java loaders (contains `MNAAS_Daily_ProcessStart`, `MNNAS_Daily_PartitionDate`, etc.). |
| **$api_month_year_ScriptName** | Shell script that performs the Sqoop export back to Oracle. |
| **$sim_inventory_SqoopLog** | Log file where all stdout/stderr and `logger` messages are appended. |
| **$HDFSSqoopDir**, **$Sim_inventory_backup** | HDFS directories used for staging and backup of imported files. |
| **$SDP_Receipient_List**, **$MOVE_DEV_TEAM**, **$SDP_ticket_from_email** | Email/SDP ticketing parameters for failure notifications. |

---

### 8. Suggested Improvements (TODO)

1. **Add a pre‑flight validation function** that checks the existence and non‑emptiness of all required variables from the property file before any processing begins. This will produce a clear early‑exit error rather than a downstream failure.

2. **Implement retry logic for critical external commands** (Sqoop import/export, Hive load, Impala refresh). A simple loop with exponential back‑off and a configurable max‑retry count would increase robustness against transient network or service hiccups.  

--- 

*End of documentation.*