**File:** `move-mediation-scripts/bin/MNAAS_Sqoop_sim_imei_history.sh`  

---

### 1. High‑Level Summary
This Bash script extracts the **SIM‑IMEI change history** from an Oracle source system via Sqoop, stages the raw CSV files in HDFS, loads them into a Hive temporary table, merges the data into the production Hive/Impala table, and updates a shared process‑status file. It includes built‑in single‑instance guarding, success/failure flag handling, logging, and automated email/SDP‑ticket generation on errors. The script is part of the daily MNAAS data‑move pipeline that processes multiple SIM‑related tables (e.g., SimInventory, SimChange).

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`sim_imei_history_table_Sqoop`** | * Orchestrates the end‑to‑end load: updates process flags, logs start, removes old HDFS files, runs the Sqoop import, sets HDFS permissions, truncates Hive temp & target tables, loads data into temp table, inserts into final table, refreshes Impala, logs success/failure. |
| **`terminateCron_successful_completion`** | * Marks the job as successful in the shared status file, logs completion, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | * Logs failure, updates run‑time stamp, triggers email/SDP notification, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | * Sends a failure notification email (and implicitly creates an SDP ticket) only once per run, updates the “email sent” flag, and logs the action. |
| **Main program block** | * Guarantees a single instance using the PID stored in the status file, checks the daily process flag (0 = idle, 1 = running), invokes the Sqoop routine, and routes to the appropriate termination function. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_sim_imei_history.properties` – defines all environment variables used (see *External Config* below). |
| **External Services** | • Oracle DB (via JDBC URL)  <br>• Hadoop HDFS (target directory)  <br>• Hive/Impala (SQL execution)  <br>• Local mail subsystem (`mail` command)  <br>• SDP ticketing system (triggered by email). |
| **Primary Input Data** | Oracle query `${sim_imei_history_table_Query}` – returns columns: `old_imei, new_imei, iccid, msisdn, imsi, old_timestamp, new_timestamp, source, result, reason, productid, business_unit_id, business_unit_name, tcl_secs_id, tcl_secs_name`. |
| **Primary Outputs** | • HDFS CSV files under `${sim_imei_history_table_Dir}`  <br>• Hive tables: `${dbname}.${sim_imei_history_temp_table_name}` (temp) and `${dbname}.${sim_imei_history_table_name}` (final)  <br>• Impala table refresh  <br>• Log file `$SIM_IMEI_History_table_SqoopLogName`  <br>• Updated process‑status file `$SIM_IMEI_History_Sqoop_ProcessStatusFile`. |
| **Side Effects** | • Overwrites/creates PID entry in status file. <br>• Alters several flag entries (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, etc.). <br>• Sends email on failure (may create an SDP ticket). |
| **Assumptions** | • All variables defined in the properties file are valid and accessible. <br>• Oracle credentials are correct and have SELECT rights on the source view/table. <br>• Hadoop, Hive, Impala services are up and reachable. <br>• The `mail` command is configured to deliver external mail. <br>• The script runs under a user with appropriate HDFS, Hive, and OS permissions. |

---

### 4. Interaction with Other Scripts & Components  

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Other MNAAS Sqoop scripts** (e.g., `MNAAS_Sqoop_sim_imei_history.sh`, `MNAAS_Sqoop_sim_imei_history_backup.sh`) | All share the same **process‑status file** (`$SIM_IMEI_History_Sqoop_ProcessStatusFile`) and follow the same flag‑based execution model, ensuring only one daily load runs at a time. |
| **Daily orchestration cron** | Typically invoked by a cron entry that runs after midnight; the script’s internal PID guard prevents overlapping runs. |
| **Hive/Impala** | Uses `hive -S -e` to truncate/load tables and `impala-shell` to refresh the target table, making the data instantly queryable for downstream analytics. |
| **SDP ticketing / Email** | Failure handling calls `email_and_SDP_ticket_triggering_step`, which sends an email to `$GTPMailId` (CC list in `$ccList`). The email body references the log file, acting as the trigger for an SDP ticket in the incident‑management system. |
| **Properties file** | Centralizes configuration; any change to DB connection, HDFS path, table names, or mail recipients must be made there, affecting all scripts that source it. |
| **Logging infrastructure** | All logs are appended to `$SIM_IMEI_History_table_SqoopLogName`; downstream monitoring tools (e.g., Splunk, ELK) can ingest this file for alerting. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Plain‑text DB credentials** in the properties file. | Store credentials in a secure vault (e.g., Hadoop KMS, HashiCorp Vault) and retrieve them at runtime; restrict file permissions to the script user. |
| **Single‑mapper Sqoop (`-m 1`)** may become a bottleneck as data volume grows. | Parameterize mapper count via a property; benchmark and increase to an appropriate parallelism level. |
| **Process‑status file corruption** could block future runs. | Implement atomic writes (e.g., `mv` a temp file) and add a health‑check that resets flags after a configurable timeout. |
| **Email spam on repeated failures** (multiple runs may resend). | Add a back‑off timer or deduplication logic based on last‑sent timestamp; ensure the “email sent” flag is persisted across runs. |
| **HDFS permission errors** (`chmod 777` is overly permissive). | Use proper HDFS ACLs; avoid world‑writable permissions. |
| **No retry logic for Sqoop import** – a transient network glitch aborts the whole pipeline. | Wrap the Sqoop command in a retry loop with exponential back‑off; capture and log each attempt. |
| **Hard‑coded paths** limit portability. | Move all absolute paths into the properties file or environment variables. |

---

### 6. Running & Debugging the Script  

1. **Preparation**  
   ```bash
   # Verify the properties file exists and is readable
   ls -l /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_sim_imei_history.properties
   ```
2. **Manual Execution** (for testing)  
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_Sqoop_sim_imei_history.sh
   ```
   - The script will automatically source the properties, check the PID guard, and start the load.  
   - Use `set -x` (already enabled) to see each command as it runs.

3. **Monitoring**  
   - Tail the log file: `tail -f $SIM_IMEI_History_table_SqoopLogName`  
   - Verify Hive tables: `hive -e "select count(*) from $dbname.$sim_imei_history_table_name;"`  
   - Check Impala refresh: `impala-shell -i $IMPALAD_HOST -q "describe $dbname.$sim_imei_history_table_name;"`

4. **Debugging Common Issues**  
   - **PID guard prevents start** – inspect the status file: `cat $SIM_IMEI_History_Sqoop_ProcessStatusFile` and kill stale PID if necessary.  
   - **Sqoop import failure** – run the generated Sqoop command manually (copy from log) to view detailed JDBC errors.  
   - **Hive load failure** – run the Hive `load data` and `insert` statements interactively to capture syntax errors.  
   - **Email not sent** – verify `mail` configuration (`/etc/mail.rc`) and that `$GTPMailId` is reachable.

5. **Exit Codes**  
   - `0` – successful completion (status file set to *Success*).  
   - `1` – failure; email/SDP ticket generated, status file set to *Failure*.

---

### 7. External Configuration & Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `SIM_IMEI_History_table_SqoopLogName` | Full path of the log file for this run. |
| `SIM_IMEI_History_Sqoop_ProcessStatusFile` | Shared status/flag file used across MNAAS daily jobs. |
| `dbname` | Hive database name. |
| `sim_imei_history_table_name` | Target Hive table (final). |
| `sim_imei_history_temp_table_name` | Temporary Hive staging table. |
| `sim_imei_history_table_Dir` | HDFS directory where Sqoop writes CSV files. |
| `OrgDetails_*` (ServerName, PortNumber, Service, Username, Password) | Oracle JDBC connection details. |
| `sim_imei_history_table_Query` | The SELECT query executed by Sqoop (must end with `AND \$CONDITIONS`). |
| `IMPALAD_HOST` | Hostname of the Impala daemon for table refresh. |
| `ccList`, `GTPMailId` | Email recipients for failure notifications. |
| `MNAAS_Sqoop_sim_imei_history_Scriptname` | Script name used in logs and status file. |
| `MNAAS_FlagValue` (derived) | Daily process flag (0 = idle, 1 = running). |

*All of the above are defined in the sourced `.properties` file; any change must be reflected there.*

---

### 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace plain‑text Oracle credentials with a key‑based retrieval (e.g., `kinit` + Kerberos or Vault) and modify the Sqoop command to use a secure password file (`--password-file`).  
2. **Add Retry & Alerting Logic** – Wrap the Sqoop import and Hive load steps in a retry loop (configurable attempts) and integrate with the existing monitoring system (e.g., send a separate alert if retries exceed threshold).  

---