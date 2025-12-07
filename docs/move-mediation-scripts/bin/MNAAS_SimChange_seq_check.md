**MNAAS_SimChange_seq_check.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`MNAAS_SimChange_seq_check.sh` is a daily orchestration script that validates the sequence integrity of SIM‑change (IMEI change) files received from two regional sources – Singapore (SNG01) and Holland (HOL01). It reads a property file for configuration, updates a shared process‑status file, invokes a Java class to perform the actual sequence‑number checks, moves processed files to an “after‑check” staging area, and generates alerts (email and SDP ticket) when missing or out‑of‑order files are detected. Successful runs update the status file to “Success”; failures trigger failure handling and ticket creation.

---

### 2. Key Functions / Logical Units

| Function / Block | Responsibility |
|------------------|----------------|
| **`seqno_check_SNG01`** | Handles the Singapore IMEI‑change file set: updates status flag, logs start, picks the oldest file, ensures only one Java process runs, calls the Java sequence‑check class, evaluates the result, sends a missing‑file email (if needed), and moves the file to the post‑check directory. |
| **`seqno_check_HOL01`** | Same as above but for the Holland file set. |
| **`terminateCron_successful_completion`** | Writes “Success” and final timestamps to the shared status file, logs completion, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, logs end time, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Marks the job as “Failure” in the status file, checks whether an SDP ticket has already been raised, and if not sends a failure email and creates an SDP ticket via `mailx`. |
| **Main Program** | Prevents concurrent runs by checking a PID stored in the status file, updates the PID, reads the current process flag, decides which region(s) to process, and finally calls the appropriate termination routine. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration** | `MNAAS_SimChange_seq_check.properties` (sourced at start). Contains paths, filenames, Java class/JAR locations, email lists, etc. |
| **External Files / Directories** | - `$MNAASMainStagingDirDaily_v1/$Singapore01_IMEIChange_extn` (incoming Singapore files) <br> - `$MNAASMainStagingDirDaily_v1/$Holland01_Actives_extn` (incoming Holland files) <br> - `$MNAAS_Actives_seq_check_ProcessStatusFilename` (shared status file) <br> - `$MNAAS_seq_check_logpath_simchange` (log file) <br> - `$MNAASMainStagingDirDaily_afr_seq_check` (post‑check staging) |
| **Java Component** | `$MNAAS_MNAAS_seqno_check_classname` (class name) executed with `$MNAAS_Main_JarPath` and `$CLASSPATHVAR`. Takes the selected filename and several history/missing‑file tracking files as arguments. |
| **Outputs** | - Updated status file (flags, timestamps, PID, job status) <br> - Log entries appended to `$MNAAS_seq_check_logpath_simchange` <br> - Possibly an email (missing file alert) <br> - Possibly an SDP ticket (failure) <br> - Files moved to the “after‑check” directory |
| **Side Effects** | - Sends email via `/bin/mail` and `/usr/bin/mailx` <br> - Creates an SDP ticket by emailing `insdp@tatacommunications.com` <br> - May spawn a Java process (resource consumption) |
| **Assumptions** | - All environment variables referenced in the property file are defined and point to existing, writable locations. <br> - Java runtime and required JARs are installed and compatible. <br> - Mail utilities (`mail`, `mailx`) are configured and can reach the internal mail system. <br> - The status file is the single source of truth for process coordination across scripts. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | How it interacts |
|---------------------|------------------|
| **Other MNAAS daily scripts** (e.g., `MNAAS_SimChange_load.sh`, `MNAAS_RawFileCount_Checks.sh`) | Share the same `$MNAAS_Actives_seq_check_ProcessStatusFilename`. The flag values set by this script determine whether downstream load scripts should run. |
| **Java Sequence‑Check Class** (`MNAAS_seqno_check`) | Performs the core validation; its success/failure drives the script’s branching. |
| **SDP Ticketing System** | Triggered via email when a failure occurs; downstream incident‑management processes rely on this ticket. |
| **Monitoring / Scheduler** (cron) | The script is invoked by a daily cron job; it writes PID and status flags to prevent overlapping runs. |
| **Mailing Lists** (`${ccList}`, `$GTPMailId`, `$MOVE_DEV_TEAM`) | Used for alerting operations and development teams. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale PID / false positive “already running”** | Job may be skipped, causing data backlog. | Implement a lock file with a timeout; verify the PID is still alive before assuming a running process. |
| **Missing or malformed property file** | Script aborts early, no processing. | Add a check after sourcing the properties; exit with a clear error and send an alert if any required variable is empty. |
| **Java class failure (non‑zero exit)** | Files are not moved; alerts may be missed. | Capture Java stdout/stderr to a separate log; include the Java error output in the failure email. |
| **Mail delivery failure** | Operators not notified of missing files or failures. | Verify mail command exit status; fallback to writing to a local “alert” file that a monitoring daemon can pick up. |
| **File system permission issues** (read/write on staging dirs) | Files cannot be processed or moved. | Pre‑run a permission check; log and abort with a distinct error code. |
| **Concurrent runs from multiple cron entries** | Race conditions, duplicate processing. | Centralize scheduling through a workflow engine (e.g., Oozie/Airflow) or enforce a single‑instance lock. |

---

### 6. Typical Execution / Debugging Steps

1. **Run Manually**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files
   ./MNAAS_SimChange_seq_check.sh
   ```
2. **Observe Log**  
   ```bash
   tail -f $MNAAS_seq_check_logpath_simchange
   ```
3. **Check Status File**  
   ```bash
   cat $MNAAS_Actives_seq_check_ProcessStatusFilename
   ```
4. **Validate PID** (if script reports “already running”)  
   ```bash
   ps -p $(grep MNAAS_Script_Process_Id $MNAAS_Actives_seq_check_ProcessStatusFilename | cut -d= -f2)
   ```
5. **Force Re‑run (after confirming no stray process)**  
   - Remove the stale PID entry from the status file or delete the lock file (if implemented).  
   - Re‑execute the script.

6. **Debug Java Call**  
   - Add `-Xlog:gc` or other JVM options in the property file.  
   - Capture Java output: modify the call to `... $MNAAS_MNAAS_seqno_check_classname ... 2>&1 | tee -a $MNAAS_seq_check_logpath_simchange`.

---

### 7. External Configuration & Environment Variables

| Variable (defined in `.properties`) | Role |
|-------------------------------------|------|
| `MNAAS_seq_check_logpath_simchange` | Path to the script’s log file. |
| `MNAAS_Actives_seq_check_ProcessStatusFilename` | Shared status/flag file. |
| `MNAASMainStagingDirDaily_v1` | Base directory for daily incoming files. |
| `Singapore01_IMEIChange_extn`, `Singapore01_Actives_extn` | Sub‑directory names for Singapore files. |
| `Holland01_Actives_extn` | Sub‑directory name for Holland files. |
| `MNAAS_Main_JarPath`, `CLASSPATHVAR` | Java classpath configuration. |
| `MNAAS_MNAAS_seqno_check_classname` | Fully‑qualified Java class to invoke. |
| `MNASS_SeqNo_Check_actives_current_missing_files` | Temp file listing missing sequences. |
| `MNASS_SeqNo_Check_actives_history_of_files`, `MNASS_SeqNo_Check_actives_missing_history_files` | Historical tracking files for the Java class. |
| `GTPMailId`, `ccList`, `MOVE_DEV_TEAM`, `SDP_ticket_from_email` | Email recipients / sender addresses. |
| `Dname_MNAAS_Actives_seqno_check` | Java process name used for duplicate‑process detection. |
| `MNAAS_FlagValue` (derived) | Determines which region(s) to process. |

---

### 8. Suggested Improvements (TODO)

1. **Replace PID‑based concurrency guard with a robust lock file** (e.g., `flock`) that automatically expires after a configurable timeout to avoid stale locks.
2. **Add explicit validation of all required environment variables** after sourcing the properties file; abort early with a clear error message if any are missing or point to non‑existent paths.

---