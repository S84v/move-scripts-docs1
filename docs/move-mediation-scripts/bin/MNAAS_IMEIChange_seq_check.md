**MNAAS_IMEIChange_seq_check.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`MNAAS_IMEIChange_seq_check.sh` is a daily orchestration script that validates the sequence integrity of IMEI‑change files received from two regional sources (Singapore 01 and Holland 01). For each source it picks the earliest pending file from the staging area, invokes a Java sequence‑check utility, records success or missing‑sequence conditions, notifies the operations team by e‑mail (and optionally raises an SDP ticket), and finally moves the processed file to an “after‑check” directory. The script maintains a shared process‑status file to coordinate with other MNAAS daily jobs and to prevent concurrent executions.

---

### 2. Key Functions / Responsibilities

| Function | Responsibility |
|----------|----------------|
| **seqno_check_SNG01** | Process the oldest IMEI‑change file for Singapore 01, run the Java sequence‑check, log results, send missing‑sequence alerts, and archive the file. |
| **seqno_check_HOL01** | Same as above but for Holland 01. |
| **terminateCron_successful_completion** | Update the process‑status file to “Success”, write final timestamps, log termination, and exit with status 0. |
| **terminateCron_Unsuccessful_completion** | Log failure, trigger e‑mail/SDP ticket creation, write “Failure” to status file, and exit with status 1. |
| **email_and_SDP_ticket_triggering_step** | Compose and send a formatted SDP ticket e‑mail (via `mailx`) if a failure has not yet generated a ticket. |
| **Main Program** (bottom of script) | Guard against concurrent runs using the PID stored in the status file, decide which source checks to run based on the `MNAAS_Daily_ProcessStatusFlag`, and invoke the appropriate functions. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_IMEIChange_seq_check.properties` – defines all environment‑specific variables (paths, filenames, Java class, mail lists, etc.). |
| **External Files / Directories** | - Staging directories: `$MNAASMainStagingDirDaily_v2/$Singapore01_imeichange_extn` and `$MNAASMainStagingDirDaily_v2/$Holland01_imeichange_extn` <br> - Archive directory: `$MNAASMainStagingDirDaily_afr_seq_check` <br> - Process‑status file: `$MNAAS_IMEIChange_seq_check_ProcessStatusFilename` <br> - Log file (append): `$MNAAS_seq_check_logpath_imeichange` <br> - History / missing‑sequence files used by Java: `$MNASS_SeqNo_Check_imeichange_history_of_files`, `$MNASS_SeqNo_Check_imeichange_missing_history_files`, `$MNASS_SeqNo_Check_imeichange_current_missing_files` |
| **Command‑line / System Calls** | - `java` execution of `$MNAAS_MNAAS_seqno_check_classname` <br> - `mail` (or `mailx`) for alert e‑mails <br> - `logger` for syslog entries |
| **Outputs** | - Updated process‑status file (flags, timestamps, PID) <br> - Log entries in `$MNAAS_seq_check_logpath_imeichange` <br> - Optional alert e‑mail with attached missing‑sequence file <br> - Optional SDP ticket e‑mail <br> - Processed file moved to archive directory |
| **Side Effects** | - May spawn a Java process that accesses HDFS / DB (depends on Java class implementation). <br> - Sends external e‑mail traffic. <br> - Updates shared status file used by other MNAAS daily scripts. |
| **Assumptions** | - All variables referenced in the properties file are defined and point to writable locations. <br> - Java class is present in `$MNAAS_Main_JarPath` and is compatible with the supplied arguments. <br> - Mail utilities (`mail`, `mailx`) are configured and can reach the recipients. <br> - The process‑status file is the single source of truth for concurrency control. |

---

### 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Pre‑decessor scripts** (e.g., `MNAAS_IMEIChange_Extract.sh`) | Populate the staging directories with raw IMEI‑change files that this script consumes. |
| **Subsequent scripts** (e.g., `MNAAS_IMEIChange_Load.sh`) | Expect the files to be present in `$MNAASMainStagingDirDaily_afr_seq_check` after successful sequence validation. |
| **Shared Process‑Status File** (`$MNAAS_IMEIChange_seq_check_ProcessStatusFilename`) | Updated by this script and read by other daily MNAAS jobs to coordinate execution order and detect failures. |
| **Java Sequence‑Check Class** (`$MNAAS_MNAAS_seqno_check_classname`) | Performs the actual sequence validation, writes missing‑sequence information to the “current missing” file, and may interact with HDFS/DB. |
| **Monitoring / Alerting** | Sends e‑mail alerts and creates SDP tickets that are consumed by the operations monitoring platform (e.g., ServiceNow or internal ticketing). |
| **Cron Scheduler** | Typically invoked from a daily cron entry; the script itself checks for an existing PID to avoid overlapping runs. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID / false positive “process already running”** | Job may be skipped, causing downstream load failures. | Add a sanity check on the PID’s start time; if older than a threshold, clear the PID entry. |
| **Missing or malformed properties file** | Script aborts early, no logs, no alerts. | Validate existence and readability of the properties file at start; fail fast with a clear message. |
| **Java class failure (non‑zero exit code)** | Missing‑sequence detection not performed; downstream jobs may ingest corrupt data. | Capture Java stderr to a dedicated log; retry logic or fallback to a “manual review” flag. |
| **Mail delivery failure** | Operators not notified of missing sequences. | Verify mail exit status; on failure, write to a local “mail‑failure” file and raise an SDP ticket. |
| **Log or status file not writable** | No audit trail; status flag may stay at “Running”. | Pre‑run permission check; if unwritable, abort and send an immediate alert via syslog. |
| **Concurrent execution of the same Java class** | Duplicate processing, race conditions on history files. | The script already checks `ps -ef | grep $Dname_MNAAS_IMEIChange_seqno_check`; ensure the grep pattern is precise (e.g., exclude the grep process itself). |
| **File naming collisions in archive directory** | Overwrite of previously processed files. | Use a timestamped sub‑directory or rename with a unique suffix before moving. |

---

### 6. Running / Debugging the Script

1. **Standard execution (cron)**  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_IMEIChange_seq_check.sh
   ```
   The script appends to the log defined in the properties file.

2. **Manual run (interactive)**  
   ```bash
   set -x   # enable tracing
   ./MNAAS_IMEIChange_seq_check.sh
   ```
   Observe the console output and the log file for detailed steps.

3. **Check current status**  
   ```bash
   cat $MNAAS_IMEIChange_seq_check_ProcessStatusFilename
   ```
   Look for `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_Script_Process_Id`, and `MNAAS_job_status`.

4. **Force re‑run (clear stale PID)**  
   ```bash
   sed -i 's/^MNAAS_Script_Process_Id=.*/MNAAS_Script_Process_Id=0/' $MNAAS_IMEIChange_seq_check_ProcessStatusFilename
   ```

5. **Debug Java component**  
   - Verify `$CLASSPATHVAR` and `$MNAAS_Main_JarPath`.  
   - Run the Java class manually with a test file to see stdout/stderr.  

6. **Validate e‑mail**  
   ```bash
   echo "test" | mail -s "MNAAS test" user@example.com
   ```
   Ensure the mail system is functional before relying on alerts.

---

### 7. External Configuration & Environment Variables

| Variable (from properties) | Role |
|----------------------------|------|
| `MNAAS_seq_check_logpath_imeichange` | Path to the script’s log file (append). |
| `MNAAS_IMEIChange_seq_check_ProcessStatusFilename` | Shared status file for PID, flags, timestamps. |
| `MNAASMainStagingDirDaily_v2` | Base staging directory for incoming files. |
| `Singapore01_imeichange_extn`, `Holland01_imeichange_extn` | Sub‑directory names for each region’s files. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Archive directory after sequence check. |
| `MNAAS_MNAAS_seqno_check_classname` | Fully‑qualified Java class name invoked for validation. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java). |
| `MNASS_SeqNo_Check_imeichange_*` | History / missing‑sequence tracking files used by Java. |
| `Dname_MNAAS_IMEIChange_seqno_check` | Java process name for concurrency guard. |
| `ccList`, `GTPMailId` | Recipient lists for missing‑sequence alert e‑mail. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Parameters for SDP ticket e‑mail generation. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath` | Java classpath components. |

All of the above must be defined in `MNAAS_IMEIChange_seq_check.properties`; the script aborts if any are missing.

---

### 8. Suggested Improvements (TODO)

1. **Robust PID handling** – Replace the ad‑hoc `ps | grep` check with `pgrep -f "$Dname_MNAAS_IMEIChange_seqno_check"` and add a timeout/age check to clean stale entries automatically.
2. **Centralised error handling** – Wrap the Java invocation and mail steps in a reusable function that captures stdout/stderr, writes to a dedicated error log, and returns a standardized exit code, reducing duplication between the two region‑specific functions. 

---