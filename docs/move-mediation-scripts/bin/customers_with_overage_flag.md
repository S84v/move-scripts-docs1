**File:** `move-mediation-scripts/bin/customers_with_overage_flag.sh`  

---

## 1. Purpose (one‑paragraph summary)  
This script orchestrates the daily “customers‑with‑overage‑flag” data load. It first refreshes a temporary Hive/Impala table, runs a Java job to populate that table with the current month’s usage data, then runs a second Java job to merge the results into the production table. Throughout the process it updates a shared process‑status file, writes detailed logs, and on failure sends an alert email (and optionally creates an SDP ticket). The script is designed to be invoked by a scheduler (e.g., cron) and includes safeguards against concurrent executions.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_temp_table** | * Truncates the temp Hive table.<br>* Executes an Impala `REFRESH` on the temp table.<br>* Launches the Java class `customers_with_overage_flag_temp_classname` to load the temp table for the current month.<br>* Refreshes Impala metadata again and logs success/failure. |
| **MNAAS_insert_into_main_table** | * Calls the Java class `customers_with_overage_flag_classname` to merge temp data into the production table.<br>* Refreshes Impala metadata for the target table and logs success/failure. |
| **terminateCron_successful_completion** | * Resets the process‑status flag to *idle* (0) and records a successful run timestamp in the status file.<br>* Writes final log entries and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | * Records the failure timestamp, logs the error, triggers alert handling, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | * Marks the job as *Failure* in the status file.<br>* Sends a pre‑formatted email to the configured distribution list (and CC list).<br>* Sets a flag to avoid duplicate ticket/email generation. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `customers_with_overage_flag.properties` – defines all environment‑specific variables (paths, DB names, class names, hostnames, email addresses, etc.). |
| **Runtime inputs** | Current date derived inside the script (`date` commands). No command‑line arguments are required. |
| **External services** | • Hive (via `hive -S -e "TRUNCATE …"`).<br>• Impala (via `impala-shell`).<br>• Java runtime (custom JARs on `$MNAAS_Main_JarPath`).<br>• Mail system (`mail` command). |
| **Outputs** | • Log file at `$MNAAS_customers_with_overage_flag_LogPath` (appended).<br>• Process‑status file (`$customers_with_overage_flag_ProcessStatusFileName`) – contains flags, PID, timestamps, email‑sent flag, etc.<br>• Updated Hive/Impala tables (temp and production). |
| **Side effects** | • Potential creation of an SDP ticket (implicit – the script only sends an email; ticket creation is assumed downstream).<br>• System‑wide process‑status flag may affect downstream scripts that read the same status file. |
| **Assumptions** | • All variables referenced in the properties file are defined and point to reachable resources.<br>• The Java classes are idempotent and can be run multiple times safely.<br>• The host running the script has network access to Hive/Impala and the mail server.<br>• Only one instance of the script runs at a time (enforced by PID check). |

---

## 4. Interaction with Other Components  

| Component | How this script connects |
|-----------|--------------------------|
| **Process‑status files** | Shared with other “MNAAS” scripts (e.g., `addon_subs_aggr_loading.sh`, `api_med_data.sh`). The flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, etc.) drive conditional execution in those scripts. |
| **Java JARs** | The classes referenced (`$customers_with_overage_flag_temp_classname`, `$customers_with_overage_flag_classname`) are part of the MNAAS codebase used by many loading scripts. |
| **Hive/Impala tables** | Temp table `${dbname}.${customers_with_overage_flag_temp_tblname}` and target table `${dbname}.${customers_with_overage_flag_tblname}` are also accessed by other aggregation scripts (e.g., `cdr_buid_search_loading.sh`). |
| **Scheduler** | Typically invoked by a cron entry (see other scripts for similar patterns). The PID guard prevents overlapping runs. |
| **Alerting pipeline** | Email sent via `mail` may be consumed by an external monitoring system that creates SDP tickets. The `MNAAS_email_sdp_created` flag prevents duplicate alerts. |
| **Logging infrastructure** | All scripts write to a common log directory (`$MNAAS_customers_with_overage_flag_LogPath`), enabling centralized log aggregation. |

---

## 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – PID check may miss a hung process. | Add a lock file with a timeout, or use `flock` to guarantee exclusive execution. |
| **Java job failure** – silent exit code not captured. | Verify that the Java classes exit with non‑zero status on error; optionally capture stdout/stderr to the log. |
| **Hive/Impala connectivity loss** – script will abort but may leave temp table truncated. | Implement a pre‑flight connectivity test; on failure, skip truncation and send alert. |
| **Missing/incorrect properties** – undefined variables cause script to fail early. | Validate required variables after sourcing the properties file; abort with a clear error if any are missing. |
| **Email delivery failure** – alerts not sent. | Check the exit status of `mail`; fallback to an alternative notification channel (e.g., Slack webhook). |
| **Log file growth** – unbounded log size over time. | Rotate logs via `logrotate` or implement size‑based truncation within the script. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites** – Ensure the properties file exists and all variables are correctly set. Verify that Hive, Impala, Java, and the mail command are reachable from the host.  
2. **Manual execution** – Run:  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./customers_with_overage_flag.sh
   ```  
   The script prints each command (`set -x`) and appends detailed logs to the configured log file.  
3. **Debugging tips**  
   * Check the process‑status file for the current flag values (`cat $customers_with_overage_flag_ProcessStatusFileName`).  
   * If the script exits early with “Process/Jar is already running”, verify that no stale PID remains (`ps -ef | grep $Dname_customers_with_overage_flag`).  
   * Inspect the Java class logs (if they write to stdout/stderr) – they are captured by the script’s `logger` calls.  
   * Use `tail -f $MNAAS_customers_with_overage_flag_LogPath` while the script runs to watch progress.  

---

## 7. External Configuration & Environment Variables  

| Variable (from properties) | Role |
|----------------------------|------|
| `MNAAS_customers_with_overage_flag_LogPath` | Path to the script’s log file. |
| `customers_with_overage_flag_ProcessStatusFileName` | Shared status file that stores flags, PID, timestamps, etc. |
| `Dname_customers_with_overage_flag` | Identifier used for the PID guard (`ps -ef | grep …`). |
| `dbname` | Hive database name containing the tables. |
| `customers_with_overage_flag_temp_tblname` / `customers_with_overage_flag_tblname` | Hive table names for temp and production data. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon used for `impala-shell`. |
| `customers_with_overage_flag_temp_tblname_refresh` / `customers_with_overage_flag_tblname_refresh` | Impala `REFRESH` statements (SQL strings). |
| `MNAAS_Main_JarPath` | Directory containing the custom Java JAR(s). |
| `CLASSPATHVAR` | Additional classpath entries for Java. |
| `customers_with_overage_flag_temp_classname` / `customers_with_overage_flag_classname` | Fully‑qualified Java class names executed for temp load and final merge. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java). |
| `MNAAS_customers_with_overage_flag_ScriptName` | Human‑readable script name used in logs/emails. |
| `MailId`, `ccList` | Email recipients for failure alerts. |
| `MNAAS_email_sdp_created` | Flag in the status file to avoid duplicate alerts. |

---

## 8. Suggested Improvements (TODO)

1. **Add explicit configuration validation** – after sourcing the properties file, loop through a required‑variables list and abort with a clear error if any are empty.  
2. **Replace the ad‑hoc PID check with `flock`** – e.g., `exec 200>/var/lock/customers_overage.lock && flock -n 200 || { logger "Lock held, exiting"; exit 1; }`. This eliminates race conditions and stale PID issues.  

---