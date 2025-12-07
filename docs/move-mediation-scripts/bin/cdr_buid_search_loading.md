**cdr_buid_search_loading.sh – High‑Level Documentation**  

---

### 1. Purpose (one‑paragraph summary)  
`cdr_buid_search_loading.sh` is a daily Hadoop‑based ETL driver that loads “CDR BUID search” data from a pre‑aggregated traffic partition file into a temporary Hive table, then merges it into the production Impala table used by the MNAAS (Mobile Network Analytics & Assurance System) reporting layer. The script orchestrates Hive truncation, Java‑based bulk loading, Impala table refresh, and maintains a process‑status file that drives idempotent execution, error handling, and automated alerting (email + SDP ticket) on failure.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_temp_table** | * Truncates the temporary Hive table (`${dbname}.${cdr_buid_seacrh_temp_tblname}`) <br> * Executes the Java bulk‑loader class `${cdr_buid_search_temp_loading_classname}` to populate the temp table from `${MNAAS_Traffic_Partitions_Daily_Uniq_FileName}` <br> * Refreshes the Impala metadata for the temp table <br> * Updates the process‑status file flag to **1** and logs success/failure |
| **MNAAS_insert_into_main_table** | * Runs the Java loader `${cdr_buid_search_loading_classname}` to insert data directly into the production table <br> * Refreshes Impala metadata for the production table <br> * Updates the process‑status file flag to **2** and logs success/failure |
| **terminateCron_successful_completion** | * Resets status flag to **0** (idle) <br> * Writes success markers (`MNAAS_job_status=Success`, run‑time, etc.) to the status file <br> * Emits final log entries and exits with code 0 |
| **terminateCron_Unsuccessful_completion** | * Writes failure run‑time to status file <br> * Calls `email_and_SDP_ticket_triggering_step` <br> * Exits with code 1 |
| **email_and_SDP_ticket_triggering_step** | * Sets `MNAAS_job_status=Failure` <br> * Sends a pre‑formatted email to `${MailId}` (CC `${ccList}`) if an SDP ticket has not yet been created (`MNAAS_email_sdp_created=No`) <br> * Flags that an SDP ticket/email has been generated to avoid duplicate alerts |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `cdr_buid_search_loading.properties` – defines all environment variables used throughout the script (paths, table names, class names, jar locations, log file, process‑status file, email recipients, etc.). |
| **External Services** | • **Hive** – for `TRUNCATE TABLE` <br>• **Impala** – `impala-shell` to refresh table metadata <br>• **Java** – bulk‑loader JAR (`${MNAAS_Main_JarPath}`) <br>• **Mail** – `mail` command for alerting <br>• **SDP ticketing** – implied external system triggered by email (no direct API call) |
| **Data Files** | `MNAAS_Traffic_Partitions_Daily_Uniq_cdr` – source CDR partition file (path defined in properties). |
| **Process‑Status File** | `${cdr_buid_search_ProcessStatusFileName}` – a plain‑text key/value file used for inter‑run coordination, PID tracking, flag management, and alert‑state. |
| **Log File** | `${MNAAS_cdr_buid_search_LogPath}` – appended with all `logger` output and error messages. |
| **Outputs** | • Populated temporary Hive table (`${cdr_buid_seacrh_temp_tblname}`) <br>• Populated production Impala table (`${cdr_buid_seacrh_tblname}`) <br>• Updated status file (flags, timestamps, job status) <br>• Email/SDP ticket on failure |
| **Assumptions** | • All variables referenced are correctly defined in the properties file. <br>• Hive, Impala, and Java runtime are reachable from the host. <br>• The process‑status file is writable and not concurrently edited by another script. <br>• Mail subsystem (`mail` command) is configured and can send to `${MailId}`. |

---

### 4. Interaction with Other Scripts / Components  

| Interaction | Description |
|-------------|-------------|
| **Properties File** (`cdr_buid_search_loading.properties`) | Shared across multiple MNAAS loading scripts (e.g., `cdr_buid_search_summary.sh`, `cdr_buid_search_*` scripts). Provides a single source of truth for table names, jar paths, and log locations. |
| **Java Loader Classes** (`${cdr_buid_search_temp_loading_classname}`, `${cdr_buid_search_loading_classname}`) | Same JAR (`${MNAAS_Main_JarPath}`) is used by other loading scripts (e.g., `Sim_Inventory_Sqoop.sh` uses a different class but same jar). |
| **Process‑Status File** (`${cdr_buid_search_ProcessStatusFileName}`) | Consumed by any downstream or retry scripts that need to know whether the temp load has completed (e.g., a nightly aggregation script that depends on the main table being up‑to‑date). |
| **Impala Refresh Queries** (`${cdr_buid_seacrh_temp_tblname_refresh}`, `${cdr_buid_seacrh_tblname_refresh}`) | These are simple `INVALIDATE METADATA` or `REFRESH` statements that other reporting jobs rely on to see the latest data. |
| **Email/SDP Ticketing** | The email body references “MNAAS_MSISDN_Level_Daily_Usage_Aggr_Failed”, indicating that downstream aggregation jobs (e.g., `billable_sub_with_charge_and_limit.sh`) monitor this flag for failure handling. |
| **Cron Scheduler** | The script is intended to be invoked by a daily cron job; other scripts in the `move-mediation-scripts/bin` directory follow the same pattern (e.g., `api_med_data.sh`). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent Execution** – PID stored in status file may become stale if the script crashes. | Duplicate loads, data corruption. | Add a lock file with a timeout; verify the PID is still alive before assuming it’s running. |
| **Missing/Incorrect Property Values** – Undefined variables cause Hive/Java failures. | Job failure, silent data gaps. | Validate required variables at script start; fail fast with clear error messages. |
| **Java Loader Failure** – Non‑zero exit code not captured correctly. | Partial data load, inconsistent state. | Capture stdout/stderr of Java process to a separate log; add retry logic for transient failures. |
| **Impala Refresh Failure** – Table metadata not refreshed. | Downstream queries see stale data. | Check `impala-shell` return code; if non‑zero, retry or abort with alert. |
| **Email/SDP Flooding** – Repeated failures could generate many tickets. | Alert fatigue. | Implement a back‑off counter in the status file (e.g., only send email if last sent > 4 h ago). |
| **Log File Growth** – Unlimited appending may fill disk. | Service outage. | Rotate logs via `logrotate`; include size‑based rotation in cron. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **1. Verify environment** | `source /app/hadoop_users/MNAAS/MNAAS_Property_Files/cdr_buid_search_loading.properties` and `env | grep MNAAS` to ensure all required vars are set. |
| **2. Dry‑run (no DB changes)** | Comment out the `hive` and `java` lines, replace them with `echo "Would run: …"` to confirm variable substitution. |
| **3. Execute manually** | `bash -x /path/to/cdr_buid_search_loading.sh` – the `-x` flag (already set via `set -x`) prints each command with expanded variables. |
| **4. Check PID handling** | `cat $cdr_buid_search_ProcessStatusFileName | grep MNAAS_Script_Process_Id` – ensure the PID matches the running process (`ps -p <pid>`). |
| **5. Review logs** | `tail -f $MNAAS_cdr_buid_search_LogPath` while the script runs. |
| **6. Verify table contents** | In Hive/Impala: `SELECT COUNT(*) FROM ${dbname}.${cdr_buid_seacrh_tblname} WHERE ...;` to confirm rows were loaded. |
| **7. Simulate failure** | Force the Java command to exit with non‑zero (`false`) and observe that the script calls `email_and_SDP_ticket_triggering_step`. |
| **8. Clean up after test** | Reset status flag: `sed -i 's/MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=0/' $cdr_buid_search_ProcessStatusFileName`. |

---

### 7. External Configuration & Environment Variables  

| Variable (from properties) | Role |
|----------------------------|------|
| `MNAAS_cdr_buid_search_LogPath` | Path to the script’s log file. |
| `cdr_buid_search_ProcessStatusFileName` | Central status/flag file for coordination. |
| `Dname_cdr_buid_search` | Identifier used to detect running Java processes (`ps -ef | grep $Dname_cdr_buid_search`). |
| `dbname` | Hive database containing the target tables. |
| `cdr_buid_seacrh_temp_tblname` / `cdr_buid_seacrh_tblname` | Temp and production table names. |
| `CLASSPATHVAR` | Additional classpath entries for Java. |
| `MNAAS_Main_JarPath` | Full path to the JAR containing loader classes. |
| `cdr_buid_search_temp_loading_classname` / `cdr_buid_search_loading_classname` | Fully‑qualified Java class names invoked for temp and main loads. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java). |
| `MNAAS_Traffic_Partitions_Daily_Uniq_FileName` | Source CDR file location. |
| `IMPALAD_HOST` | Hostname of the Impala daemon used for `impala-shell`. |
| `cdr_buid_seacrh_temp_tblname_refresh` / `cdr_buid_seacrh_tblname_refresh` | Impala refresh statements (e.g., `INVALIDATE METADATA <tbl>`). |
| `MNAAS_cdr_buid_search_loading_ScriptName` | Human‑readable script identifier used in logs/emails. |
| `MailId`, `ccList` | Email recipients for failure alerts. |
| `MNAAS_cdr_buid_search_loading_ScriptName` (duplicate) | Used in termination messages. |

---

### 8. Suggested Improvements (TODO)

1. **Lock File with Timeout** – Replace PID‑only check with a lock file (`flock` or custom) that automatically expires after a configurable period to avoid stale locks.
2. **Centralised Error Handling Library** – Extract the repetitive status‑file updates, logging, and email logic into a shared shell library (`mnaas_common.sh`) used by all loading scripts to reduce duplication and ensure consistent behavior. 

--- 

*End of documentation.*