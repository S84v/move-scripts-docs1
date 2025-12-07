**billable_sub_with_charge_and_limit.sh – High‑Level Documentation**  

---

### 1. Summary  
`billable_sub_with_charge_and_limit.sh` is a daily Hadoop‑based ETL driver that loads **billable subscription data with charge limits** into MNAAS (Mobile Network Analytics & Assurance System). It orchestrates two stages:  

1. **Temp‑table load** – truncates a staging Hive table, refreshes its Impala metadata, and runs a Java class that extracts the current day’s raw data into the temp table.  
2. **Main‑table load** – invokes a second Java class that aggregates the temp data (including previous‑month context) and writes the results into the production fact table.  

The script maintains a lightweight process‑status file (key‑value pairs) to coordinate runs, prevent parallel execution, and expose success/failure flags to downstream monitoring tools. Logging, alerting (email), and optional SDP ticket creation are performed on failure.  

---

### 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_insert_into_temp_table`** | *Stage 1* – Truncates the staging Hive table, refreshes Impala metadata, runs the Java “temp‑loading” class for the current day, validates exit code, and updates the status file. |
| **`MNAAS_insert_into_main_table`** | *Stage 2* – Executes the Java “main‑loading” class with parameters for the current year‑month, current day, and previous month (YYYYMM). Refreshes Impala metadata on success and updates the status file. |
| **`terminateCron_successful_completion`** | Writes a *Success* state to the status file, resets the process flag, logs completion, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Writes a *Failure* timestamp, logs the error, and exits with code 1 (does **not** invoke email/ticket – that is delegated to `email_and_SDP_ticket_triggering_step`). |
| **`email_and_SDP_ticket_triggering_step`** | Sends a failure notification email (and optionally creates an SDP ticket) the first time a failure is detected; subsequent failures are suppressed to avoid flooding. |
| **Main script block** | Guard against concurrent runs using the PID stored in the status file, decide which stages to execute based on the `MNAAS_Daily_ProcessStatusFlag`, and drive the overall flow. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `billable_sub_with_charge_and_limit_loading.properties` – defines all variables referenced in the script (DB names, table names, class names, process‑status file path, log path, mail recipients, etc.). |
| **Environment variables** | Implicitly used: `CLASSPATHVAR`, `IMPALAD_HOST`, `Dname_billable_sub_with_charge_and_limit`, `MNAAS_Property_filename`. Must be exported before the script runs (usually by the properties file). |
| **External services** | - **Hive** (via `hive -S -e`) – truncates staging table.<br>- **Impala** (via `impala-shell`) – refreshes table metadata.<br>- **Java runtime** – runs two classes from `mnaas_prod_billing.jar`.<br>- **Mail daemon** (`mail` command) – sends failure alerts.<br>- **File system** – reads/writes the process‑status file and log file. |
| **Inputs** | - Current date (derived inside script).<br>- Previous month identifier (`prev_mon_yr`).<br>- All parameters passed to the Java classes (property file path, dates, log path). |
| **Outputs** | - Populated **temp** Hive table (`${dbname}.${billable_sub_with_charge_and_limit_temp_tblname}`).<br>- Populated **main** fact table (`${dbname}.${billable_sub_with_charge_and_limit_tblname}`).<br>- Log file (`$MNAAS_billable_sub_with_charge_and_limit_LogPath`).<br>- Updated process‑status file (flags, PID, timestamps). |
| **Side Effects** | - Potentially long‑running Java processes that consume Hadoop cluster resources.<br>- Email sent on failure.<br>- Process‑status file mutation (used by monitoring/other scripts). |
| **Assumptions** | - The properties file exists and contains valid values for every variable used.<br>- Hive/Impala services are reachable from the host running the script.<br>- The Java JAR and class names are compatible with the current schema.<br>- Only one instance of the script runs per day (enforced by PID check). |
| **Dependencies on other scripts** | - Other MNAAS loading scripts follow the same status‑file convention; monitoring dashboards may read the same file to display job health.<br>- Cron scheduler (or an orchestrator like Oozie/Airflow) invokes this script daily. |

---

### 4. Connection to Other Components  

| Component | How this script interacts |
|-----------|---------------------------|
| **`billable_sub_with_charge_and_limit_loading.properties`** | Provides all runtime parameters (DB names, table names, class names, mail lists, status‑file path). |
| **Process‑status file** (`$billable_sub_with_charge_and_limit_ProcessStatusFileName`) | Shared with other MNAAS loading scripts; the flag values (`0`, `1`, `2`) indicate stage progress and are read/written by this script and possibly by monitoring tools. |
| **Java JAR** (`mnaas_prod_billing.jar`) | Contains the two loader classes invoked here; the same JAR is used by other “*_loading.sh” scripts (e.g., `Move_Invoice_Register.sh`). |
| **Hive/Impala tables** | Staging (`*_temp_tblname`) and production (`*_tblname`) tables are also referenced by downstream reporting jobs (e.g., Tableau data extracts). |
| **Alerting pipeline** | Failure email uses `$MailId`, `$ccList` defined in the properties file; SDP ticket creation is a hook that may be consumed by the ITSM system. |
| **Scheduler** | Typically launched by a daily cron entry (`/etc/cron.d/mnaas` or similar). The PID guard prevents overlapping runs, which is a pattern used across the whole “move‑mediation‑scripts” suite. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID guard may fail if the status file is corrupted or the previous PID is stale. | Duplicate loads, data corruption, resource contention. | Add a sanity check on the PID’s start time (`ps -p $PID -o lstart=`) and purge stale entries after a configurable timeout. |
| **Java job failure without proper exit code** – If the Java class hangs or exits non‑zero but the script does not capture it, downstream steps may run on incomplete data. | Inconsistent aggregates, downstream reporting errors. | Enforce a timeout wrapper (`timeout` command) around the Java calls and verify the existence of expected output rows. |
| **Impala metadata refresh failure** – If the `impala-shell -q "REFRESH ..." ` fails, the next query may read stale data. | Stale or missing rows in the main table. | Capture the exit status of `impala-shell` and abort on non‑zero, logging the error. |
| **Mail delivery failure** – `mail` command may silently drop messages if the MTA is down. | No alert raised, failure goes unnoticed. | Add a fallback to write the alert to a file and/or trigger an HTTP webhook to a monitoring system. |
| **Hard‑coded date logic** – Uses `date -d "$(date +%Y-%m-1) -1 month"` which may break on the first day of the month in some locales. | Wrong `prev_mon_yr` value, causing incorrect aggregation. | Use GNU `date` with explicit timezone and locale, or compute previous month via a small utility script with unit tests. |
| **Log file growth** – Continuous `exec 2>>` appends without rotation. | Disk exhaustion. | Configure logrotate for the path `$MNAAS_billable_sub_with_charge_and_limit_LogPath`. |

---

### 6. Running & Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Prerequisite** | Ensure the properties file `billable_sub_with_charge_and_limit_loading.properties` is present and exported variables are correct. |
| **Manual execution** | ```bash\nset -x   # optional, already in script\n./billable_sub_with_charge_and_limit.sh\n``` |
| **Dry‑run (no DB writes)** | Comment out the `java` lines or replace them with `echo "Would run Java …"`; also add `--quiet` to `impala-shell` to avoid refresh. |
| **Check status** | ```bash\ncat $billable_sub_with_charge_and_limit_ProcessStatusFileName\n``` |
| **View logs** | ```bash\ntail -f $MNAAS_billable_sub_with_charge_and_limit_LogPath\n``` |
| **Force re‑run of a specific stage** | Edit the status file flag to `0` (run both stages) or `2` (run only main stage) and re‑execute. |
| **Debug Java class** | Run the Java command directly with `-Xdebug` or `-verbose:gc` to capture JVM logs; verify classpath (`$CLASSPATHVAR:$MNAAS_Main_JarPath`). |
| **PID troubleshooting** | ```bash\nps -ef | grep $Dname_billable_sub_with_charge_and_limit\n``` to see stray processes. |
| **Email verification** | Ensure the local MTA can send mail: `echo test | mail -s "test" $MailId`. Check `/var/log/maillog` for delivery status. |

---

### 7. External Config / Environment Variables  

| Variable (populated by properties) | Purpose |
|-----------------------------------|---------|
| `MNAAS_billable_sub_with_charge_and_limit_LogPath` | Absolute path to the script’s log file. |
| `billable_sub_with_charge_and_limit_ProcessStatusFileName` | Path to the key‑value status file used for coordination. |
| `dbname` | Hive/Impala database name. |
| `billable_sub_with_charge_and_limit_temp_tblname` | Staging table name (temp). |
| `billable_sub_with_charge_and_limit_tblname` | Production fact table name. |
| `billable_sub_with_charge_and_limit_temp_tblname_refresh` | Impala REFRESH statement for the temp table. |
| `billable_sub_with_charge_and_limit_tblname_refresh` | Impala REFRESH statement for the main table. |
| `billable_sub_with_charge_and_limit_temp_loading_classname` | Fully‑qualified Java class for temp load. |
| `billable_sub_with_charge_and_limit_loading_classname` | Fully‑qualified Java class for main load. |
| `Dname_billable_sub_with_charge_and_limit` | Identifier used for the Java `-Dname` system property and PID guard. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java). |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. |
| `MailId`, `ccList` | Recipients for failure notifications. |
| `MNAAS_billable_sub_with_charge_and_limit_loading_ScriptName` | Human‑readable script name used in logs/emails. |
| `CLASSPATHVAR` | Additional classpath entries required by the Java runtime. |

*If any of the above variables are missing or empty, the script will abort with a non‑zero exit code; verify the properties file before scheduling.*

---

### 8. Suggested TODO / Improvements  

1. **Add robust PID management** – Replace the ad‑hoc `ps | grep` check with a lock file (`flock`) or a proper PID file that validates the process is still alive and not a stale entry.  
2. **Centralize status handling** – Extract the status‑file read/write logic into a reusable library (e.g., `status_util.sh`) so that all MNAAS loading scripts share a single, tested implementation and can emit JSON for modern monitoring tools.  

--- 

*End of documentation.*