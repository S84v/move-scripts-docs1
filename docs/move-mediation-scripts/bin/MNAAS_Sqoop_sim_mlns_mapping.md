**File:** `move-mediation-scripts/bin/MNAAS_Sqoop_sim_mlns_mapping.sh`  

---

## 1. Purpose (one‑paragraph summary)

This script extracts the **SIM‑to‑MLNS (Mobile Line Number Service) mapping** data from an Oracle source system via Sqoop, stages the raw files in HDFS, and loads them into Hive/Impala tables used by downstream mediation processes. It coordinates execution through a shared status file, logs progress, refreshes the Impala metadata, and on failure sends an email and creates an SDP ticket. The script is intended to run daily (or as part of a larger batch) under cron, ensuring only one instance runs at a time.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `sim_mlns_mapping_table_Sqoop` | *Main ETL step*: cleans HDFS target directory, runs Sqoop import, sets HDFS permissions, truncates Hive staging & target tables, loads raw files into the staging table, merges with reference tables, refreshes Impala metadata, and logs success/failure. |
| `terminateCron_successful_completion` | Updates the shared status file to indicate a successful run, logs termination, and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, updates status file with failure timestamp, triggers email/SDP ticket, and exits with code 1. |
| `email_and_SDP_ticket_triggering_step` | Sends a failure notification email (with CC list) and marks that an SDP ticket has been created to avoid duplicate alerts. |
| **Main program block** | Checks the PID stored in the status file to avoid concurrent runs, updates the PID, validates the daily‑process flag, invokes the ETL function, and routes to the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_sim_mlns_mapping.properties`. Expected variables (examples): <br>• `SIM_MLNS_Mapping_table_SqoopLogName` – path to the log file.<br>• `SIM_MLNS_Mapping_Sqoop_ProcessStatusFile` – shared status file.<br>• `dbname`, `sim_mlns_mapping_table_name`, `sim_mlns_mapping_temp_table_name`, `sim_mlns_mapping_table_Dir` – Hive/Impala identifiers and HDFS directory.<br>• Oracle connection details (`OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password`).<br>• `sim_mlns_mapping_table_Query` – the Sqoop free‑form query.<br>• `IMPALAD_HOST`, `ccList`, `GTPMailId` – Impala host and email settings. |
| **External Services** | • Oracle DB (source).<br>• Hadoop HDFS (target directory).<br>• Hive/Impala (tables).<br>• Local mail subsystem (`mail` command). |
| **Primary Input Data** | Rows returned by the Oracle query defined in the properties file. |
| **Primary Output Data** | Populated Hive tables: `dbname.sim_mlns_mapping_temp_table_name` (staging) and `dbname.sim_mlns_mapping_table_name` (final). Impala metadata refreshed for the final table. |
| **Side Effects** | • HDFS directory cleanup (`hadoop fs -rm -r`).<br>• Log file appends.<br>• Status file updates (process flag, PID, timestamps, job status).<br>• Email sent on failure.<br>• Potential SDP ticket creation (outside script). |
| **Assumptions** | • The properties file exists and contains all required variables.<br>• The script runs on a node with Hadoop, Hive, Impala, and Sqoop client installed and correctly configured.<br>• The Oracle credentials are valid and have SELECT rights for the query.<br>• The shared status file is writable by the script user and is used consistently by other MNAAS scripts for coordination. |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Shared Status File** (`SIM_MLNS_Mapping_Sqoop_ProcessStatusFile`) | Used by multiple MNAAS scripts (e.g., `MNAAS_SimInventory_seq_check.sh`, `MNAAS_SimInventory_tbl_Load.sh`) to coordinate daily process flags, PID tracking, and overall job status. |
| **Log File** (`SIM_MLNS_Mapping_table_SqoopLogName`) | Consolidated log consumed by monitoring/alerting tools and referenced by other scripts for troubleshooting. |
| **Hive Tables** (`$dbname.$sim_mlns_mapping_table_name`, `$dbname.$sim_mlns_mapping_temp_table_name`) | Populated here; downstream mediation or reporting scripts read these tables (e.g., monthly status scripts, invoice generation scripts). |
| **Impala Refresh** | Ensures that any downstream Impala‑based queries see the latest data immediately. |
| **Email/SDP Notification** | Failure notifications may be aggregated with other script alerts; the `MNAAS_email_sdp_created` flag prevents duplicate tickets across the suite. |
| **Potential Caller** | Typically invoked by a cron entry or a master orchestration script that runs the full MNAAS data‑load pipeline. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Credential Exposure** – Oracle password stored in plain text in the properties file. | Store credentials in a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and source them at runtime; restrict file permissions to the script user only. |
| **Single‑Mapper Sqoop (`-m 1`)** – May become a bottleneck for large data volumes. | Evaluate data size; increase mapper count (`-m N`) after testing, or enable `--direct` mode if supported. |
| **HDFS Directory Deletion** – `hadoop fs -rm -r ${sim_mlns_mapping_table_Dir}/*` removes all files, potentially wiping data if the Sqoop step fails. | Perform a “dry‑run” check, or move old files to a backup location before deletion. |
| **No Retry Logic** – Failure of any command aborts the whole run. | Wrap critical commands (Sqoop, Hive load) in retry loops with exponential back‑off; log each attempt. |
| **Email Flooding** – Repeated failures could generate many emails. | Enforce a throttling mechanism (e.g., only send one email per hour) and ensure the `MNAAS_email_sdp_created` flag is reliably reset after ticket resolution. |
| **Process Overlap** – PID check relies on a shared status file; stale PID could block new runs. | Add a sanity check on the PID’s age (e.g., if process > 24 h, treat as stale and clear). |
| **Hard‑coded Paths** – Absolute paths reduce portability. | Parameterise base directories in the properties file. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure Hadoop, Hive, Impala, and Sqoop clients are in the `$PATH`.  
   - Verify the properties file exists and is readable:  
     ```bash
     . /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_sim_mlns_mapping.properties
     echo $SIM_MLNS_Mapping_table_SqoopLogName
     ```  
   - Confirm the script user has write permission on the log file and status file.

2. **Manual Execution** (for testing)  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts/
   bash -x MNAAS_Sqoop_sim_mlns_mapping.sh   # -x enables shell tracing
   ```

3. **Typical Cron Entry** (example)  
   ```cron
   0 2 * * * /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_Sqoop_sim_mlns_mapping.sh >> /var/log/mnaas/cron.log 2>&1
   ```

4. **Debugging Tips**  
   - Check the generated log file (`$SIM_MLNS_Mapping_table_SqoopLogName`) for timestamps and error messages.  
   - Verify the PID stored in the status file matches the running process (`ps -p <pid>`).  
   - Run individual commands manually (e.g., the Sqoop import) to isolate failures.  
   - Use Hive’s `SHOW CREATE TABLE` to confirm table schemas match the expected columns.  
   - After a failure, inspect the email sent and the `MNAAS_email_sdp_created` flag to avoid duplicate alerts.

---

## 7. External Configurations & Environment Variables

| Variable (from properties) | Role |
|----------------------------|------|
| `SIM_MLNS_Mapping_table_SqoopLogName` | Path to the script‑specific log file. |
| `SIM_MLNS_Mapping_Sqoop_ProcessStatusFile` | Shared status/flag file used for coordination. |
| `dbname` | Hive/Impala database name. |
| `sim_mlns_mapping_table_name` | Final Hive table name. |
| `sim_mlns_mapping_temp_table_name` | Staging Hive table name. |
| `sim_mlns_mapping_table_Dir` | HDFS directory where Sqoop writes raw files. |
| `OrgDetails_*` (ServerName, PortNumber, Service, Username, Password) | Oracle connection details for Sqoop. |
| `sim_mlns_mapping_table_Query` | Free‑form SQL query executed by Sqoop. |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `ccList`, `GTPMailId` | Email recipients for failure notifications. |
| `MNAAS_Sqoop_sim_mlns_mapping_Scriptname` | Script name used in status updates and logs (usually set in the properties file). |

---

## 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace plain‑text Oracle credentials with a secret‑management solution and source them at runtime.  
2. **Add Retry & Idempotency** – Implement a generic retry wrapper for Sqoop and Hive commands, and move the HDFS cleanup step to occur *after* a successful import to avoid data loss on transient failures.  

---