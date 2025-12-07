**High‑Level Documentation – `move-mediation-scripts/bin/usage_overage_charge.sh`**

---

### 1. Purpose (one‑paragraph summary)

`usage_overage_charge.sh` is a nightly orchestration wrapper that builds the *usage‑overage charge* data set for the MNAAS billing data‑warehouse. It first truncates and repopulates a temporary Hive/Impala table using a Java loader, then merges the temporary data into the production table. The script maintains a process‑status file to guarantee single‑instance execution, to record progress flags, and to trigger failure notifications (email + SDP ticket) when a step fails. It is typically invoked by a cron job and forms the final “charge‑generation” step of the daily billing pipeline.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_insert_into_temp_table`** | *Step 1* – Truncates the temp table, runs an Impala refresh, executes the Java class that extracts raw usage data for the current day and writes it into the temp table. Updates the status file flag to **1**. |
| **`MNAAS_insert_into_main_table`** | *Step 2* – Calls the Java class that aggregates the temp data, applies overage logic, and inserts the result into the production charge table. Updates the status file flag to **2**. |
| **`terminateCron_successful_completion`** | Resets the status file to **0** (idle), marks job as *Success*, records run‑time, logs completion, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Records failure time, logs the error, invokes (optionally) `email_and_SDP_ticket_triggering_step`, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Sends a failure email to the configured distribution list, creates an SDP ticket flag in the status file, and logs the action. (Called only from the failure path.) |
| **Main script block** | Checks for an existing PID (single‑instance guard), reads the current flag from the status file, decides which steps to run (temp → main, or only main if flag = 2), and drives the overall flow. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration file** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/usage_overage_charge.properties` – defines all variables referenced in the script (paths, table names, class names, log file, status file, email lists, etc.). |
| **Environment variables** | `IMPALAD_HOST`, `CLASSPATHVAR`, `Dname_usage_overage_charge` – must be exported before execution (usually by the properties file or a wrapper). |
| **External services** | - **Hive** (via `hive -S -e`) – truncates temp table.<br>- **Impala** (via `impala-shell`) – refreshes table metadata.<br>- **Java** (via `java -cp …`) – runs two loader classes (`usage_overage_charge_temp_classname` and `usage_overage_charge_classname`).<br>- **Mail** (via `mail`) – sends failure notifications.<br>- **SDP ticketing** – not directly invoked; only a flag is set for downstream processes. |
| **Data stores** | - Temp Hive table `${dbname}.${usage_overage_charge_temp_tblname}`.<br>- Production charge table `${dbname}.${usage_overage_charge_tblname}` (implicitly referenced in the refresh query). |
| **Process‑status file** | `$usage_overage_charge_ProcessStatusFileName` – a plain‑text key/value file used for: <br>• PID tracking (`MNAAS_Script_Process_Id`).<br>• Current step flag (`MNAAS_Daily_ProcessStatusFlag`).<br>• Job status (`MNAAS_job_status`).<br>• Email/SDP flag (`MNAAS_email_sdp_created`). |
| **Log file** | `$MNAAS_usage_overage_charge_LogPath` – appended with all `logger` output and Java STDERR. |
| **Outputs** | - Populated production overage‑charge table.<br>- Updated status file (Success/Failure, timestamps).<br>- Log file entry.<br>- Optional email + SDP ticket flag on failure. |
| **Assumptions** | - The properties file supplies *all* required variables; missing entries cause immediate failure.<br>- Hive/Impala services are reachable from the host running the script.<br>- Java class JAR (`mnaas_prod_billing.jar`) is compatible with the current schema.<br>- Only one instance runs at a time (enforced by PID check). |

---

### 4. Interaction with Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **`runMLNS.sh`**, **`service_based_charges.sh`**, **`traffic_nodups_summation.py`**, etc. | Follow the same pattern: status‑file guard, temp‑table load, main‑table merge. They share the same *process‑status* directory and may be scheduled sequentially in the nightly pipeline. |
| **Cron scheduler** | Typically invoked from a nightly cron entry (e.g., `0 2 * * * /path/usage_overage_charge.sh`). The script’s PID guard prevents overlapping runs if a previous night’s job overruns. |
| **Downstream reporting / billing** | The production charge table is consumed by downstream billing jobs (e.g., invoice generation, revenue assurance). Any failure here propagates as missing overage charges. |
| **Alerting pipeline** | The `email_and_SDP_ticket_triggering_step` sets the `MNAAS_email_sdp_created` flag; downstream monitoring scripts poll this flag to raise tickets in the SDP system. |
| **Shared libraries** | The JAR (`mnaas_prod_billing.jar`) is also used by other billing scripts (e.g., `service_based_charges.sh`). Changes to the JAR affect all callers. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID guard fails (e.g., stale PID) | Duplicate loads, data corruption | Add a sanity check that the PID truly belongs to a running `usage_overage_charge.sh` process; clean stale PID entries after a timeout. |
| **Java class failure** – non‑zero exit code | Incomplete charge data, downstream billing errors | Capture Java stdout/stderr to a separate file; implement retry logic for transient failures (e.g., network hiccup). |
| **Hive/Impala connectivity loss** | Table truncation or refresh fails, job aborts | Pre‑flight health check (`impala-shell -q "SELECT 1"`); alert on repeated failures. |
| **Status‑file corruption** (e.g., manual edit) | Wrong flag values, script may skip steps or loop indefinitely | Store a checksum (e.g., MD5) of the status file after each write and verify at start. |
| **Log file growth** | Disk exhaustion on the node | Rotate logs daily via `logrotate`; enforce a max size. |
| **Email/SDP flood** – repeated failures generate many alerts | Alert fatigue | Debounce email notifications (e.g., only send once per hour) and rely on the `MNAAS_email_sdp_created` flag. |
| **Missing/incorrect properties** | Immediate script exit, no processing | Validate required variables at script start; fail fast with a clear error message. |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file exists and contains all required keys (search for `usage_overage_charge_` prefixes).  
   - Export any environment variables not defined in the properties file (`IMPALAD_HOST`, `CLASSPATHVAR`, `Dname_usage_overage_charge`).  
   - Verify Hive/Impala connectivity (`hive -e "show databases"`; `impala-shell -i $IMPALAD_HOST -q "show tables"`).  

2. **Manual execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts/bin
   ./usage_overage_charge.sh   # run as the same user that cron uses
   ```
   - Check the log file (`$MNAAS_usage_overage_charge_LogPath`) for progress.  
   - Inspect the status file (`$usage_overage_charge_ProcessStatusFileName`) to confirm flag transitions (0 → 1 → 2 → 0).  

3. **Debug mode**  
   - The script already runs with `set -x` (command tracing).  
   - To capture Java output, modify the Java command lines to redirect stdout/stderr to a separate file, e.g.:  
     ```bash
     java ... $usage_overage_charge_temp_classname ... > /tmp/overage_temp.log 2>&1
     ```  
   - Use `ps -ef | grep usage_overage_charge` to verify no stray processes remain after a failure.  

4. **Simulating a failure**  
   - Force the Java class to exit with a non‑zero code (e.g., `exit 1` inside the class) and observe that the script writes the failure flag, sends the email, and sets `MNAAS_email_sdp_created=Yes`.  

5. **Post‑run validation**  
   - Query the production table to ensure rows were inserted for the expected date range:  
     ```sql
     SELECT COUNT(*) FROM ${dbname}.${usage_overage_charge_tblname}
     WHERE charge_date = CURRENT_DATE - INTERVAL 1 DAY;
     ```

---

### 7. External Config / Environment Dependencies

| Item | Description | Usage |
|------|-------------|-------|
| `usage_overage_charge.properties` | Centralised key/value file (path hard‑coded). Contains: <br>• `MNAAS_usage_overage_charge_LogPath` (log file) <br>• `usage_overage_charge_ProcessStatusFileName` (status file) <br>• `usage_overage_charge_temp_tblname`, `usage_overage_charge_tblname` (Hive tables) <br>• `usage_overage_charge_temp_tblname_refresh`, `usage_overage_charge_tblname_refresh` (Impala REFRESH statements) <br>• `usage_overage_charge_temp_classname`, `usage_overage_charge_classname` (Java class names) <br>• `MNAAS_usage_overage_charge_ScriptName`, `MailId`, `ccList` (email settings) | Sourced at script start (`.`). All subsequent variables are read from here. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. | Used by `impala-shell -i`. |
| `CLASSPATHVAR` | Additional classpath entries for Java. | Combined with the JAR path for `java -cp`. |
| `Dname_usage_overage_charge` | Identifier used for the Java `-Dname` system property and for the PID guard. | Prevents concurrent Java executions. |
| `MNAAS_Main_JarPath` (hard‑coded) | Path to `mnaas_prod_billing.jar`. | Provides the Java loader classes. |
| `MailId`, `ccList` | Email recipients for failure alerts. | Used by `mail` command. |

---

### 8. Suggested Improvements (TODO)

1. **Add explicit validation of required properties** – before any processing, loop through a list of mandatory keys and abort with a clear error if any are missing or empty.  
2. **Implement a retry wrapper for the Java steps** – on transient failures (e.g., network hiccup to Hive/Impala), automatically retry up to 3 times with exponential back‑off, logging each attempt.  

---