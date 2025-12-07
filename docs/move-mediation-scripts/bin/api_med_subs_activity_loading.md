**File:** `move-mediation-scripts/bin/api_med_subs_activity_loading.sh`  

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily load of *API Mediation Subscriber Activity* data into the MNAAS data‑warehouse. It first truncates a staging Hive table, runs a Java loader to populate the temporary table, refreshes the Impala metadata, then runs a second Java loader to merge the temporary data into the production table and refreshes Impala again. Throughout the run it updates a shared *process‑status* file, writes detailed logs, enforces a single‑instance lock, and on failure raises an SDP ticket and sends an email alert.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_temp_table** | * Sets process flag = 1 (temp load). <br>* Ensures no other instance of the Java loader is running. <br>* Truncates the Hive temp table. <br>* Executes Java class `${api_med_subs_activity_temp_loading_classname}` to fill the temp table. <br>* On success, runs `impala-shell` refresh query `${api_med_subs_activity_temp_tblname_refresh}`. |
| **MNAAS_insert_into_main_table** | * Sets process flag = 2 (main load). <br>* Checks for concurrent Java loader. <br>* Executes Java class `${api_med_subs_activity_loading_classname}` to merge temp → main. <br>* On success, runs `impala-shell` refresh query `${api_med_subs_activity_tblname_refresh}`. |
| **terminateCron_successful_completion** | * Resets status file flags to “Success”, clears running PID, writes final log entries and exits 0. |
| **terminateCron_Unsuccessful_completion** | * Updates run‑time stamp, logs failure, calls `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | * Marks job status = Failure in status file. <br>* If an SDP ticket/email has not yet been created (`MNAAS_email_sdp_created=No`), sends a pre‑formatted email to `${MailId}` (CC `${ccList}`) and flips the flag to `Yes`. |
| **Main script block** | * Reads previous PID from status file and aborts if a prior instance is still alive. <br>* Writes current PID to status file. <br>* Determines current flag (0 = idle, 1 = temp‑only, 2 = main‑only) and drives the appropriate load functions. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/api_med_subs_activity.properties` – defines all environment variables used (paths, class names, DB hosts, email lists, etc.). |
| **Process‑status file** | `${api_med_subs_activity_ProcessStatusFileName}` – a plain‑text key/value file that stores PID, flags, timestamps, email‑sent flag, etc. Updated throughout the run. |
| **Log file** | `${MNAAS_api_med_subs_activity_LogPath}` – appended with `logger -s` statements and error output (`exec 2>>`). |
| **Hive/Impala tables** | *Temp table*: `${dbname}.${api_med_subs_activity_temp_tblname}` (truncated then loaded). <br>*Main table*: `${dbname}.${api_med_subs_activity_tblname}` (populated via Java loader). |
| **Java JAR** | `${MNAAS_Main_JarPath}` – contains the loader classes referenced by `${api_med_subs_activity_temp_loading_classname}` and `${api_med_subs_activity_loading_classname}`. |
| **External services** | - Hive metastore (via `hive -S -e`). <br>- Impala daemon (`impala-shell -i $IMPALAD_HOST`). <br>- SMTP server (used by `mail`). |
| **Side effects** | - Alters Hive/Impala tables (data load). <br>- Sends email / creates SDP ticket on failure. <br>- Updates shared status file (used by other scripts to coordinate runs). |
| **Assumptions** | - The properties file exists and defines all required variables. <br>- Hive, Impala, and the Java classpath are reachable from the host. <br>- Only one instance of the script runs at a time (enforced via PID check). <br>- The status file is writable by the script user. |

---

## 4. Interaction with Other Scripts / Components  

| Component | Relationship |
|-----------|--------------|
| **MNAAS_move_files_from_staging.sh** (previous script) | Likely stages raw activity files into HDFS / staging directories that the Java loader reads. This script assumes those files are already present. |
| **MNAAS_Weekly_KYC_Feed_Loading.sh** (parallel feed) | Independent feed loader that also updates the same status file format; the shared status file prevents overlapping runs across different feeds. |
| **Common status‑file library** | All MNAAS loading scripts read/write the same `${api_med_subs_activity_ProcessStatusFileName}` (or analogous files) to coordinate flags and PID. |
| **Java loader classes** | Implement the actual parsing/transform logic; they are invoked from this script and are shared across daily/weekly pipelines. |
| **Monitoring / Alerting** | The email/SDP step integrates with the telecom’s incident‑management system; other orchestration tools (e.g., Oozie, Airflow) may trigger this script via cron. |

---

## 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Stale PID lock** – script crashes leaving PID in status file → subsequent runs blocked. | Add a health‑check that validates the PID is still alive (`ps -p $PID`). If not, clear the lock before proceeding. |
| **Concurrent Java loader** – race condition if another process spawns the same Java class. | The script already checks `ps -ef | grep $Dname_api_med_subs_activity`. Ensure `$Dname_api_med_subs_activity` is unique per loader. |
| **Hive/Impala connectivity failure** – load stops, status file left in intermediate flag. | Wrap Hive/Impala commands in retry loops with exponential back‑off; on repeated failure, trigger alert and abort. |
| **Email/SDP flood** – repeated failures could generate many tickets. | The status flag `MNAAS_email_sdp_created` prevents duplicate alerts; ensure the flag is cleared only after a successful run. |
| **Properties file drift** – missing or malformed variables cause silent failures. | Validate required variables at script start; exit with clear error if any are undefined. |
| **Data corruption** – truncate temp table before confirming source files are ready. | Add a pre‑load validation step (e.g., file count, checksum) before truncating. |

---

## 6. Running / Debugging the Script  

**Typical execution (via cron):**  
```bash
# Cron entry (example, runs daily at 02:30)
30 2 * * * /app/hadoop_users/MNAAS/MNAAS_Property_Files/api_med_subs_activity_loading.sh >> /dev/null 2>&1
```

**Manual run (for testing):**  
```bash
# Ensure environment is clean
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/api_med_subs_activity.properties
# Run with trace enabled (already set by `set -x` in script)
bash -x /path/to/api_med_subs_activity_loading.sh
```

**Debugging steps:**  

1. **Check the log file** – `${MNAAS_api_med_subs_activity_LogPath}` contains timestamps and `logger` messages.  
2. **Inspect the status file** – grep for `MNAAS_Daily_ProcessStatusFlag` and `MNAAS_Script_Process_Id` to see current state.  
3. **Validate Java loader** – run the Java class manually with the same arguments to see stack traces.  
4. **Confirm Hive/Impala** – execute the refresh queries directly in `hive` / `impala-shell` to ensure they succeed.  
5. **PID verification** – `ps -p <PID>` should show the running script; if not, clear the PID entry.  

---

## 7. External Config / Environment Variables  

| Variable (populated in properties file) | Purpose |
|------------------------------------------|---------|
| `MNAAS_api_med_subs_activity_LogPath` | Absolute path to the script’s log file. |
| `api_med_subs_activity_ProcessStatusFileName` | Path to the shared status‑key/value file. |
| `Dname_api_med_subs_activity` | Identifier used to detect running Java loader processes. |
| `CLASSPATHVAR` | Additional classpath entries for the Java command. |
| `MNAAS_Main_JarPath` | Full path to the JAR containing loader classes. |
| `api_med_subs_activity_temp_loading_classname` | Fully‑qualified Java class for temp table load. |
| `api_med_subs_activity_temp_tblname_refresh` | Impala `INVALIDATE METADATA` or `REFRESH` statement for temp table. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. |
| `api_med_subs_activity_loading_classname` | Java class for main table merge. |
| `api_med_subs_activity_tblname_refresh` | Impala refresh statement for the production table. |
| `api_med_subs_activity_loading_ScriptName` | Human‑readable name used in logs/emails. |
| `ccList`, `MailId` | Email recipients for failure notifications. |
| `dbname`, `api_med_subs_activity_temp_tblname`, `api_med_subs_activity_tblname` | Hive database and table identifiers. |

If any of these are missing, the script will fail early (e.g., `java` will receive empty class name). Adding a validation block at the top of the script is recommended.

---

## 8. Suggested Improvements (TODO)

1. **Add upfront validation of required variables** – loop through a list of expected env vars and abort with a clear message if any are empty.  
2. **Implement a retry wrapper for Hive/Impala commands** – encapsulate `hive -S -e` and `impala-shell` calls in a function that retries up to N times with back‑off, improving resilience to transient service outages.  

---