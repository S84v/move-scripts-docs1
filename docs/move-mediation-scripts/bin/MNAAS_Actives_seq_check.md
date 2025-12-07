**File:** `move-mediation-scripts/bin/MNAAS_Actives_seq_check.sh`  

---

## 1. Purpose (one‑paragraph summary)

The script validates the sequence‑number integrity of daily “Actives” feed files received in the MNAAS (Mobile Number Activation & Allocation System) staging area. For each configured feed source it picks the newest file, invokes a Java utility that checks for missing sequence numbers, logs the result, moves the file to a post‑check directory, and raises an email/SDP ticket if gaps are detected. It also updates a shared process‑status file so downstream MNAAS jobs can safely coordinate execution, and guarantees that only a single instance runs at any time.

---

## 2. Key Functions / Responsibilities

| Function | Responsibility |
|----------|----------------|
| **seqno_check** | Loops over all feed sources defined in `FEED_FILE_PATTERN`. For each source it logs start, counts files, selects the newest file, ensures the Java seq‑no checker is not already running, executes the Java class, interprets its exit code, sends alert email (with missing‑seq file attached) if gaps exist, and moves the processed file to the “after‑seq‑check” directory. |
| **terminateCron_successful_completion** | Writes a *success* state to the shared process‑status file (`MNAAS_Actives_seq_check_ProcessStatusFilename`), records run time, logs completion, and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, calls `email_and_SDP_ticket_triggering_step`, records end‑time, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | Updates the status file to *Failure*, checks whether an SDP ticket has already been raised, and if not sends a formatted ticket request via `mailx` (including required metadata) and marks the ticket‑created flag. |
| **Main program (bottom block)** | Guarantees single‑instance execution using the PID stored in the status file, updates the PID, checks the daily process flag, invokes `seqno_check`, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Actives_seq_check.properties` – defines all environment variables used throughout the script (paths, filenames, Java class, email lists, etc.). |
| **External services** | • Java class (`$MNAAS_MNAAS_seqno_check_classname`) executed via `$CLASSPATHVAR:$MNAAS_Main_JarPath`.<br>• System logger (`logger`).<br>• Mail utilities (`mail`, `mailx`). |
| **File system inputs** | • Staging directory `$MNAASMainStagingDirDaily_v1` containing feed files matching patterns in associative array `FEED_FILE_PATTERN`.<br>• Process‑status file `$MNAAS_Actives_seq_check_ProcessStatusFilename` (shared with other MNAAS scripts). |
| **File system outputs** | • Log file `$MNAAS_seq_check_logpath_Actives` (appended).<br>• “Missing‑seq” report file `$MNASS_SeqNo_Check_actives_current_missing_files` (created/truncated).<br>• Processed feed file moved to `$MNAASMainStagingDirDaily_afr_seq_check`.<br>• Updated process‑status file (flags, PID, timestamps). |
| **Side effects** | • Sends alert email to `$GTPMailId` (and CC list).<br>• Sends SDP ticket request to `insdp@tatacommunications.com` via `mailx`.<br>• May create a new PID entry in the status file. |
| **Assumptions** | • All variables referenced in the properties file are defined and point to accessible locations.<br>• Java class returns `0` on success and non‑zero on failure.<br>• The host has `mail`, `mailx`, `logger`, `ps`, `sed`, `truncate`, `ls`, `wc`, `head`, `tail` utilities available.<br>• The process‑status file is writable by the script user and is the single source of truth for coordination. |

---

## 4. Interaction with Other Components

| Component | How this script connects |
|-----------|--------------------------|
| **MNAAS feed ingestion pipeline** | Consumes raw feed files placed by upstream ingestion jobs (e.g., `MNAAS_Actives_load_driver.sh`). |
| **Java seq‑no checker** | Invoked as `$MNAAS_MNAAS_seqno_check_classname` with arguments: property file, current filename, history files, missing‑seq file, log path. The Java class updates the missing‑seq file used for alerts. |
| **Process‑status file** | Shared with downstream jobs (e.g., `MNAAS_Actives_load.sh`). The flags `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, etc., are read/written here to coordinate start/stop and error handling. |
| **Alerting / ticketing system** | Sends email via `mail` and creates an SDP ticket via `mailx`. The ticket metadata references the same script name, enabling downstream monitoring dashboards. |
| **Cron scheduler** | Typically executed as a daily cron job; the script’s PID guard prevents overlapping runs. |
| **Log aggregation** | Appends to `$MNAAS_seq_check_logpath_Actives`, which is likely collected by a central log shipper (e.g., Splunk, ELK). |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale PID / false‑positive “already running”** | Periodically verify that the PID stored in the status file still corresponds to a live process; add a timeout check (e.g., if process > X hours, clear PID). |
| **Missing or malformed properties file** | Validate required variables at script start; abort with clear error if any are undefined. |
| **Java class failure (non‑zero exit)** | Capture Java stdout/stderr to a dedicated log; consider retry logic or fallback to a “skip” mode after a configurable number of attempts. |
| **Email or SDP ticket delivery failure** | Check exit status of `mail`/`mailx`; on failure, write to a “failed‑alerts” file and raise a monitoring alarm. |
| **File loss during move** | Use `mv -n` (no‑overwrite) and verify the destination file exists; optionally copy then delete after checksum verification. |
| **Unbounded log growth** | Rotate `$MNAAS_seq_check_logpath_Actives` via logrotate or a custom size‑based rotation script. |
| **Concurrent execution of Java checker** | The script already checks `ps` count; ensure the Java class itself is thread‑safe and does not rely on shared temp files. |

---

## 6. Running / Debugging the Script

1. **Manual execution**  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   bash MNAAS_Actives_seq_check.sh
   ```
   The script already runs with `set -x`, so each command is echoed to stdout/stderr and also logged to `$MNAAS_seq_check_logpath_Actives`.

2. **Check prerequisites**  
   ```bash
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Actives_seq_check.properties
   echo "$MNAASMainStagingDirDaily_v1"
   ls -l "$MNAASMainStagingDirDaily_v1"
   ```

3. **Validate PID guard**  
   ```bash
   grep MNAAS_Script_Process_Id "$MNAAS_Actives_seq_check_ProcessStatusFilename"
   ps -p <pid>
   ```

4. **Force a failure path (for testing alerts)**  
   - Remove or corrupt the Java class jar, or set `MNAAS_FlagValue` to a non‑zero value that triggers the error branch.
   - Observe email/SDP ticket generation and log entries.

5. **Log inspection**  
   ```bash
   tail -f "$MNAAS_seq_check_logpath_Actives"
   ```

6. **Debug missing‑seq file**  
   ```bash
   cat "$MNASS_SeqNo_Check_actives_current_missing_files"
   ```

---

## 7. External Config / Environment Variables (referenced)

| Variable | Origin | Usage |
|----------|--------|-------|
| `MNAAS_seq_check_logpath_Actives` | properties file | Path for script log file (append). |
| `MNAAS_Actives_seq_check_ProcessStatusFilename` | properties file | Shared status file (flags, PID, timestamps). |
| `FEED_FILE_PATTERN` | properties file (associative array) | Mapping of feed source → filename glob pattern. |
| `MNAASMainStagingDirDaily_v1` | properties file | Directory where raw Actives files land. |
| `MNAASMainStagingDirDaily_afr_seq_check` | properties file | Destination after successful/failed seq‑check. |
| `Dname_MNAAS_Actives_seqno_check` | properties file | Process name used for ps‑based lock. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `MNAAS_MNAAS_seqno_check_classname` | properties file | Java execution environment. |
| `MNAAS_Property_filename` | properties file | Passed to Java class (likely same as this properties file). |
| `MNASS_SeqNo_Check_actives_history_of_files`, `MNASS_SeqNo_Check_actives_missing_history_files`, `MNASS_SeqNo_Check_actives_current_missing_files` | properties file | Files used by Java class to store history and missing‑seq reports. |
| `ccList`, `GTPMailId` | properties file | Email recipients for alerts. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | properties file | Metadata for SDP ticket generation. |
| `insdp@tatacommunications.com` | hard‑coded | SDP ticket inbox. |

---

## 8. Suggested Improvements (TODO)

1. **Add explicit validation of required variables** – early‑exit with a clear message if any critical variable (e.g., staging dir, status file, Java class) is undefined or points to a non‑existent path.  
2. **Implement log rotation / size check** within the script (or ensure external logrotate) to prevent uncontrolled growth of `$MNAAS_seq_check_logpath_Actives`.  

---