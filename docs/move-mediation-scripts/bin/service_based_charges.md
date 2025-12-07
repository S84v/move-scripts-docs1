**High‑Level Summary**  
`service_based_charges.sh` is a nightly orchestration script that loads *service‑based charge* data into the MNAAS data‑warehouse. It first truncates and repopulates a temporary Hive/Impala table using a Java loader, then merges the temporary data into the production table. The script maintains a process‑status file to coordinate retries, enforce single‑instance execution, and drive downstream notifications (email/SDP ticket) on failure.

---

## 1. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_temp_table** | *Pre‑load* – Truncates the temp table, runs an Impala refresh, executes the Java class `${service_based_charges_temp_classname}` to populate the temp table for the current day. Updates status flag = 1. |
| **MNAAS_insert_into_main_table** | *Final load* – Executes the Java class `${service_based_charges_classname}` to merge temp data into the production table for the current month (and previous‑month reference). Refreshes the target table via Impala. Updates status flag = 2. |
| **terminateCron_successful_completion** | Writes *Success* metadata (flags, timestamps, job status) to the status file, logs completion, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Writes *Failure* timestamp, logs the error, and exits 1. (Optionally calls `email_and_SDP_ticket_triggering_step` – currently commented out.) |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email and marks an SDP ticket as created (guarded by a flag to avoid duplicate alerts). |

---

## 2. Execution Flow (Main Script)

1. **Singleton guard** – Reads `MNAAS_Script_Process_Id` from the status file; if a process with that PID is still running, the script aborts.  
2. **Status flag handling** – Reads `MNAAS_Daily_ProcessStatusFlag` (0 = idle, 1 = temp‑load pending, 2 = main‑load pending).  
3. **Branching**  
   * Flag 0 or 1 → run `MNAAS_insert_into_temp_table` → `MNAAS_insert_into_main_table`.  
   * Flag 2 → run only `MNAAS_insert_into_main_table`.  
4. **Completion** – On success, call `terminateCron_successful_completion`; on any error, call `terminateCron_Unsuccessful_completion`.  

All steps log to `${MNAAS_service_based_charges_LogPath}` and update `${service_based_charges_ProcessStatusFileName}`.

---

## 3. Inputs, Outputs & Side Effects  

| Category | Item | Description |
|----------|------|-------------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/service_based_charges.properties` | Defines all environment variables used (paths, DB names, class names, hostnames, email lists, etc.). |
| **Runtime Variables** | `MNAAS_service_based_charges_LogPath` | File path for script‑level logging (stderr redirected). |
| | `service_based_charges_ProcessStatusFileName` | Plain‑text status file (key=value) used for inter‑script coordination and monitoring. |
| | `Dname_service_based_charges` | Java process name used for PID checks. |
| | `dbname`, `service_based_charges_temp_tblname`, `service_based_charges_tblname_refresh` | Hive/Impala database and table identifiers. |
| | `IMPALAD_HOST` | Impala daemon host for `impala-shell` commands. |
| | `CLASSPATHVAR`, `MNAAS_Main_JarPath` | Java classpath and JAR location. |
| | `service_based_charges_temp_classname`, `service_based_charges_classname` | Fully‑qualified Java class names for temp and main loaders. |
| | `ccList`, `MailId` | Email recipients for failure alerts. |
| **External Services** | Hive (`hive -S -e`) | Executes `TRUNCATE TABLE`. |
| | Impala (`impala-shell`) | Refreshes table metadata after loads. |
| | Java (`java -cp …`) | Runs the data‑loading JARs. |
| | `mail` command | Sends failure notifications. |
| **Outputs** | Populated Hive/Impala tables (`${service_based_charges_temp_tblname}`, `${service_based_charges_tblname}`) | Ready for downstream analytics. |
| | Updated status file (flags, timestamps, job status) | Consumed by other MNAAS scripts (e.g., monitoring dashboards, downstream batch jobs). |
| | Log file (`$MNAAS_service_based_charges_LogPath`) | Auditable execution trace. |
| **Side Effects** | Potentially long‑running Java processes; consumes Hadoop cluster resources. |
| | May trigger SDP ticket creation on repeated failures (via `email_and_SDP_ticket_triggering_step`). |

---

## 4. Integration Points  

| Component | Interaction |
|-----------|-------------|
| **Other MNAAS batch scripts** (e.g., `runMLNS.sh`, `runGBS.sh`) | Share the same status‑file convention (`MNAAS_*_ProcessStatusFlag`, `MNAAS_job_status`). Coordination is achieved by reading/writing these flags. |
| **Monitoring/Alerting system** | Reads the status file and log to surface job health; failure emails feed into the IPX ticketing pipeline. |
| **Data Warehouse** | Writes to Hive/Impala tables that are later consumed by reporting/analytics jobs (e.g., `product_status_report.sh`). |
| **Java Loader JARs** | Implement the actual transformation logic; any change to class names or JAR location must be reflected in the properties file. |
| **Scheduler (cron/oozie)** | The script is invoked by a nightly cron entry; the singleton guard prevents overlapping runs. |
| **SDP ticketing** | `email_and_SDP_ticket_triggering_step` writes a flag to avoid duplicate tickets; external ticketing system consumes the email. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID check may miss a hung process (e.g., zombie). | Duplicate loads, data corruption. | Add a timeout/heartbeat check; kill stale PID after configurable grace period. |
| **Java loader failure** – Non‑zero exit code not captured if the Java process spawns background threads. | Incomplete data, silent failures. | Ensure Java class returns proper exit status; wrap call in `set -o pipefail` and capture logs. |
| **Impala metadata refresh failure** – If `impala-shell` returns error, script proceeds. | Stale table metadata, downstream queries see old data. | Check return code of each `impala-shell` command; abort on failure. |
| **Hard‑coded date arithmetic** – Uses `date -d` which may behave differently on non‑GNU `date`. | Wrong month/year for previous‑month load. | Use portable date library (e.g., `python -c` or `perl`) or validate on target OS. |
| **Email/SDP flood** – Repeated failures could generate many alerts. | Alert fatigue. | Implement exponential back‑off or a daily alert limit; ensure flag file is respected. |
| **Missing/invalid properties** – Script aborts silently if a variable is undefined. | Unclear failure mode. | Add validation block after sourcing properties (e.g., `: ${MNAAS_service_based_charges_LogPath?Missing}`) and log missing keys. |

---

## 6. Running & Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **1. Verify environment** | `source /app/hadoop_users/MNAAS/MNAAS_Property_Files/service_based_charges.properties` and run `env | grep MNAAS` to ensure all required vars are set. |
| **2. Dry‑run (no DB writes)** | Comment out the `java` and `impala-shell` lines, replace with `echo "Would run …"` to confirm flag handling and logging. |
| **3. Execute** | `bash /path/to/service_based_charges.sh` (or via cron). The script logs to `$MNAAS_service_based_charges_LogPath`. |
| **4. Check singleton** | `cat $service_based_charges_ProcessStatusFileName | grep MNAAS_Script_Process_Id` – confirm PID matches `ps -p <pid>`. |
| **5. Inspect status file** | Verify flags after run: `grep MNAAS_Daily_ProcessStatusFlag $service_based_charges_ProcessStatusFileName`. |
| **6. Debug failures** | Tail the log file while running: `tail -f $MNAAS_service_based_charges_LogPath`. Look for “failed” messages and the exit code of the Java command (`echo $?`). |
| **7. Force re‑run** | Reset flag to 0 (`sed -i 's/MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=0/' $service_based_charges_ProcessStatusFileName`). Ensure no stale PID remains. |
| **8. Unit‑test Java loader** | Run the Java class directly with test arguments to verify it can connect to Hive/Impala and write to the temp table. |

---

## 7. External Config / Environment Variables  

| Variable (from properties) | Purpose | Typical Source |
|----------------------------|---------|----------------|
| `MNAAS_service_based_charges_LogPath` | Path for script log file (stderr). | Defined per environment (dev/qa/prod). |
| `service_based_charges_ProcessStatusFileName` | Shared status/coordination file. | Central location under `/app/hadoop_users/MNAAS/...`. |
| `Dname_service_based_charges` | Identifier used for `ps` grep to detect running Java process. | Arbitrary string, must be unique across MNAAS jobs. |
| `dbname` | Hive database containing charge tables. | Configured per tenant. |
| `service_based_charges_temp_tblname` / `service_based_charges_tblname_refresh` | Table names for temp and main loads. | Hive/Impala table identifiers. |
| `IMPALAD_HOST` | Hostname/IP of Impala daemon. | Cluster configuration. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath` | Java classpath and JAR location. | Deployed under `/app/hadoop_users/MNAAS/jars/`. |
| `service_based_charges_temp_classname`, `service_based_charges_classname` | Fully‑qualified Java class names for loaders. | Part of the JAR. |
| `ccList`, `MailId` | Email recipients for failure alerts. | Ops/Support distribution list. |
| `MNAAS_service_based_charges_ScriptName` | Human‑readable script identifier used in logs/emails. | Usually the script filename. |

If any of these are missing or point to non‑existent resources, the script will fail early; add validation after the `.properties` source line.

---

## 8. Suggested Improvements (TODO)

1. **Add robust error handling for external commands** – capture and act on non‑zero exit codes from `impala-shell` and `hive`. Example:  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "$service_based_charges_temp_tblname_refresh" || { logger "Impala refresh failed"; terminateCron_Unsuccessful_completion; }
   ```
2. **Introduce a configuration validation block** – after sourcing the properties file, loop through required variables and abort with a clear message if any are unset. This prevents silent failures caused by missing env vars.  

--- 

*End of documentation.*