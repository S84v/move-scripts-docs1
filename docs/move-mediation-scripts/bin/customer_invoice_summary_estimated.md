**File:** `move-mediation-scripts\bin\customer_invoice_summary_estimated.sh`  

---

## 1. Purpose (One‑paragraph summary)

This script orchestrates the daily generation of the *Customer Invoice Summary – Estimated* dataset. It first truncates and repopulates a temporary Hive/Impala table via a Java job, then merges the temporary data into the production table using a second Java job. Execution is guarded by a process‑status file that records the current step, PID, and overall job health, ensuring only one instance runs at a time. On success the status file is reset; on failure an email alert (and optional SDP ticket) is sent.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_temp_table** | *Step 1*: Truncate the temp table, refresh Impala metadata, launch the Java class that builds the temporary invoice‑summary data for the current day. Updates the status file flag to **1**. |
| **MNAAS_insert_into_main_table** | *Step 2*: Run the Java class that inserts the temp data into the final production table (adds month‑year partition). Updates the status file flag to **2**. |
| **terminateCron_successful_completion** | Resets the status file to *idle* (`Flag=0`), records success timestamp, writes final log entries, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Records failure timestamp, writes error log entries, and exits with code 1. (Optionally calls `email_and_SDP_ticket_triggering_step` – currently commented out.) |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email (and marks an SDP ticket as created) the first time a failure occurs for a given run. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `customer_invoice_summary_estimated.properties` – defines all environment‑specific variables (log path, DB names, table names, Java class names, JAR locations, Impala host, email recipients, etc.). |
| **External Services** | - **Hive** (`hive -S -e "TRUNCATE …"`).<br>- **Impala** (`impala-shell`).<br>- **Java** (two classes executed via `$CLASSPATHVAR:$MNAAS_Main_JarPath`).<br>- **Mail** (`mail` command). |
| **Files** | - Process‑status file (`$customer_invoice_summary_estimated_ProcessStatusFileName`).<br>- Log file (`$MNAAS_customer_invoice_summary_estimated_LogPath`). |
| **Outputs** | - Populated temporary table (`${dbname}.${customer_invoice_summary_estimated_temp_tblname}`).<br>- Updated production table (`${customer_invoice_summary_estimated_tblname_refresh}`).<br>- Updated status file (flags, PID, timestamps, job status).<br>- Log entries (STDOUT/STDERR). |
| **Side Effects** | - Potential creation of an SDP ticket (via external ticketing system – not shown in script).<br>- Email alerts on failure. |
| **Assumptions** | - All variables are correctly defined in the properties file.<br>- Hive/Impala services are reachable and the user has required privileges.<br>- Java JAR contains the referenced classes and they are compatible with the current schema.<br>- The process‑status file exists and is writable.<br>- Mail utility is configured on the host. |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Properties File** | Shared across many MNAAS scripts; defines common variables (e.g., `$IMPALAD_HOST`, `$CLASSPATHVAR`). |
| **Java Classes** | `customer_invoice_summary_estimated_temp_classname` and `customer_invoice_summary_estimated_classname` are compiled in the same JAR used by other “customer_invoice_summary” scripts (e.g., `customer_invoice_summary.sh`). |
| **Process‑Status Files** | Follow the same naming convention (`*_ProcessStatusFileName`) as other MNAAS jobs, enabling a central monitoring dashboard. |
| **Logging** | Writes to a common log directory (`$MNAAS_customer_invoice_summary_estimated_LogPath`) that is aggregated by log‑collection tools (e.g., Splunk, ELK). |
| **Email/SDP** | Uses the same email template and SDP ticket flag logic as other failure‑handling scripts (e.g., `billable_sub_with_charge_and_limit.sh`). |
| **Cron Scheduler** | Typically invoked from a daily cron entry; the script’s PID guard prevents overlapping runs, a pattern used across the suite. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Concurrent Execution** – PID guard may fail if the status file is corrupted or the previous process hangs. | Duplicate loads, data corruption. | Implement a lock file with `flock` and add a timeout check on the PID’s existence. |
| **Missing/Invalid Config** – Undefined variables cause command failures. | Job aborts, no data loaded. | Validate required variables at script start; exit with clear error if any are empty. |
| **Hive/Impala Failure** – Truncate or refresh may fail silently if not captured. | Incomplete data, stale partitions. | Capture exit codes of Hive/Impala commands and abort with explicit log messages. |
| **Java Job Failure** – Non‑zero exit code only logged; no retry. | Data not generated. | Add a configurable retry loop with exponential back‑off for transient Java failures. |
| **Log/Status File Permission Issues** – Unable to write logs or update status. | No visibility, subsequent runs think job is still running. | Ensure directory permissions are set correctly; monitor disk space. |
| **Email/SDP Flooding** – Repeated failures could generate many alerts. | Alert fatigue. | Throttle email alerts (e.g., only once per hour) and ensure the “email created” flag is respected. |

---

## 6. Running / Debugging the Script

1. **Manual Invocation**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files
   . customer_invoice_summary_estimated.properties   # load env vars
   /path/to/move-mediation-scripts/bin/customer_invoice_summary_estimated.sh
   ```
   The script runs with `set -x`, so each command is echoed to the log file.

2. **Check Status Before Run**  
   ```bash
   grep -E 'MNAAS_Script_Process_Id|MNAAS_Daily_ProcessStatusFlag' \
        $customer_invoice_summary_estimated_ProcessStatusFileName
   ```

3. **Force Re‑run (e.g., after a failure)**  
   - Reset the status file manually: set `MNAAS_Daily_ProcessStatusFlag=0` and clear the PID line.  
   - Ensure no stray Java process is still running (`ps -ef | grep $Dname_customer_invoice_summary_estimated`).

4. **Debugging Tips**  
   - Tail the log while the job runs: `tail -f $MNAAS_customer_invoice_summary_estimated_LogPath`.  
   - Verify Hive/Impala commands independently: run the generated SQL statements manually.  
   - Capture Java stdout/stderr by adding `-verbose:class` or redirecting inside the function if needed.

5. **Cron Integration**  
   - Typical cron entry (example):  
     ```cron
     02 02 * * * /app/hadoop_users/MNAAS/MNAAS_Property_Files/customer_invoice_summary_estimated.sh >> /dev/null 2>&1
     ```

---

## 7. External Config / Environment Variables

| Variable (from properties) | Usage |
|----------------------------|-------|
| `MNAAS_customer_invoice_summary_estimated_LogPath` | Log file destination. |
| `customer_invoice_summary_estimated_ProcessStatusFileName` | Central status/flag file. |
| `Dname_customer_invoice_summary_estimated` | Name used for PID detection (`ps -ef | grep …`). |
| `IMPALAD_HOST` | Target Impala daemon for metadata refresh. |
| `dbname` | Hive database containing temp table. |
| `customer_invoice_summary_estimated_temp_tblname` | Temp table to truncate/populate. |
| `customer_invoice_summary_estimated_temp_tblname_refresh` | Impala `INVALIDATE METADATA` statement. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath` | Java classpath & JAR location. |
| `customer_invoice_summary_estimated_temp_classname` | Java class that builds the temp dataset. |
| `customer_invoice_summary_estimated_classname` | Java class that merges temp into main table. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java). |
| `customer_invoice_summary_estimated_tblname_refresh` | Impala refresh for the final table. |
| `MNAAS_customer_invoice_summary_estimated_ScriptName` | Human‑readable script identifier for logs/emails. |
| `MailId`, `ccList` | Recipients for failure notification. |
| `MNAAS_job_status`, `MNAAS_email_sdp_created` | Flags stored in the status file. |

---

## 8. Suggested Improvements (TODO)

1. **Add Robust Parameter Validation** – At script start, loop through a whitelist of required variables and abort with a clear error if any are empty or point to non‑existent files/directories.

2. **Replace `ps | grep` PID Check with `flock`** – Use `flock -n /tmp/customer_invoice_summary_estimated.lock` to acquire an exclusive lock; this avoids race conditions and eliminates reliance on process‑name greps.  

--- 

*End of documentation.*