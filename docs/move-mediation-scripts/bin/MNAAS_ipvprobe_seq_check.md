**File:** `move-mediation-scripts/bin/MNAAS_ipvprobe_seq_check.sh`  

---

## 1. High‑Level Summary
This Bash driver validates the daily sequence numbers of *ipvprobe* feed files that land in the Mediation staging area. For each configured feed source it picks the earliest file, invokes a Java “seq‑no‑check” class to verify continuity, records any missing sequence numbers, notifies the operations team (email + SDP ticket) when gaps are detected, and finally moves the processed file to an “after‑seq‑check” directory. The script maintains a shared *process‑status* file to coordinate concurrent runs, logs all activity, and exits with a success or failure flag that downstream jobs (e.g., loading scripts) read.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **seqno_check** | Loops over all feed sources defined in `FEED_FILE_PATTERN`. For each source it: <br>• Logs start time.<br>• Determines if files exist.<br>• Calls the Java seq‑no‑check class with the first (oldest) file.<br>• On success, inspects a temporary “missing‑seq” file: if non‑empty, emails the team with the list and moves the source file to `$MNAASMainStagingDirDaily_afr_seq_check`; otherwise just moves the file.<br>• On Java failure or if the Java process is already running, logs and triggers failure handling. |
| **terminateCron_successful_completion** | Updates the shared process‑status file to indicate *Success* (reset flag, set job status, timestamp) and writes final log entries before exiting with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, calls `email_and_SDP_ticket_triggering_step`, writes end‑time, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Marks the job status as *Failure* in the status file, checks whether an SDP ticket/email has already been generated, and if not: <br>• Sends a plain‑text alert mail to the ops list.<br>• Sends a formatted SDP ticket via `mailx` to the internal ticketing mailbox.<br>• Updates the status file to record that an email/ticket has been created. |
| **Main Program (PID guard & orchestration)** | Prevents overlapping runs by checking the PID stored in the status file. If no prior instance is alive, it writes its own PID, validates the daily flag (0 or 1), invokes `seqno_check`, and finally calls the appropriate termination routine. If a prior instance is still running, it logs and exits cleanly (treated as successful). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_ipvprobe_seq_check.properties` – defines all environment‑specific variables: <br>• `$MNAAS_seq_check_logpath_ipvprobe` (log file)<br>• `$MNAAS_ipvprobe_seq_check_ProcessStatusFilename` (shared status file)<br>• `$MNAASMainStagingDirDaily_v1` (incoming staging dir)<br>• `$MNAASMainStagingDirDaily_afr_seq_check` (post‑check dir)<br>• `FEED_FILE_PATTERN` associative array (file‑name glob per source)<br>• Java classpath, main JAR, class name, property file name, etc.<br>• Email lists (`ccList`, `GTPMailId`), SDP ticket parameters (`SDP_ticket_from_email`, `MOVE_DEV_TEAM`). |
| **Runtime inputs** | Files matching each `FEED_FILE_PATTERN` that exist under `$MNAASMainStagingDirDaily_v1`. |
| **Outputs** | • Updated *process‑status* file (flags, timestamps, job status).<br>• Log entries appended to `$MNAAS_seq_check_logpath_ipvprobe`.<br>• Optional email with missing‑seq attachment.<br>• Optional SDP ticket via `mailx`.<br>• Processed source file moved to `$MNAASMainStagingDirDaily_afr_seq_check`. |
| **Side effects** | • Starts a Java process (`java -Dname=…`).<br>• Writes/overwrites temporary files: `$MNASS_SeqNo_Check_ipvprobe_current_missing_files`, `$MNASS_SeqNo_Check_ipvprobe_history_of_files`, `$MNASS_SeqNo_Check_ipvprobe_missing_history_files`.<br>• Calls external commands: `logger`, `mail`, `mailx`, `ps`, `sed`, `truncate`, `mv`. |
| **Assumptions** | • The properties file exists and is syntactically correct.<br>• The Java class `MNAAS_MNAAS_seqno_check_classname` is compiled, reachable via `$CLASSPATHVAR:$MNAAS_Main_JarPath`, and returns exit 0 on success.<br>• Mail utilities (`mail`, `mailx`) are configured and can reach the internal mail server.<br>• The shared status file is writable by the script user.<br>• Directory paths are mounted and have sufficient permissions. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Mediation ingestion scripts** (e.g., `MNAAS_create_customer_cdr_traffic_files_*.sh`) | Produce the ipvprobe feed files that land in `$MNAASMainStagingDirDaily_v1`. |
| **Downstream loading scripts** (e.g., `MNAAS_ipvprobe_daily_recon_loading.sh`, `MNAAS_ipvprobe_monthly_recon_loading.sh`) | Expect the files to be present in `$MNAASMainStagingDirDaily_afr_seq_check` after successful seq‑check. They may also read the same *process‑status* file to verify that the seq‑check succeeded. |
| **Shared process‑status file** (`$MNAAS_ipvprobe_seq_check_ProcessStatusFilename`) | Used by multiple orchestration scripts to coordinate execution windows and to surface job status to monitoring dashboards. |
| **Java seq‑no‑check library** (`MNAAS_MNAAS_seqno_check_classname`) | Central business logic for detecting missing sequence numbers; may be shared with other feed‑type checks. |
| **Monitoring / Alerting system** | Consumes the log file and the status file; may trigger alerts if `MNAAS_job_status=Failure`. |
| **SDP ticketing system** | Receives tickets via the `mailx` command; the ticket content references the log and status file paths. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **PID‑guard race condition** (stale PID left in status file) | Script may abort erroneously, causing a missed daily check. | Implement a PID‑age check (e.g., ignore PIDs older than a configurable threshold) and/or use a lockfile (`flock`). |
| **Java class failure or missing JAR** | No seq‑check performed; downstream loads may ingest out‑of‑order data. | Add explicit validation of `$CLASSPATHVAR` and JAR existence before invoking Java; fail fast with clear log message. |
| **Mail or SDP ticket delivery failure** | Operators not notified of missing sequences. | Capture mail exit codes; retry a configurable number of times; fallback to writing a file in a “alerts” directory that a monitoring agent watches. |
| **Log file growth** | Disk exhaustion on the host. | Rotate logs via `logrotate` or implement size‑based truncation within the script. |
| **Missing or malformed properties** | Script crashes early, leaving status flag unchanged. | Validate required variables after sourcing the properties file; abort with a distinct error code and log. |
| **File move errors (e.g., permission, NFS stale handle)** | Source files remain in staging area, causing re‑processing loops. | Check exit status of `mv`; on failure, log and raise an alert; optionally copy instead of move as a fallback. |

---

## 6. Running / Debugging the Script  

1. **Standard execution** – The script is scheduled via cron (e.g., `0 2 * * * /app/hadoop_users/MNAAS/.../MNAAS_ipvprobe_seq_check.sh`). Ensure the cron user has read/write access to all configured paths.  

2. **Manual run** –  
   ```bash
   export MNAAS_ENV=prod   # if the properties file uses this variable
   /app/hadoop_users/MNAAS/MNAAS_ipvprobe_seq_check.sh
   ```
   Observe console output (the script already uses `set -x` for trace).  

3. **Debugging steps**  
   * Verify the properties file loads: `source /path/to/MNAAS_ipvprobe_seq_check.properties && env | grep MNAAS_`.  
   * Check that the status file contains a sane PID and flag: `cat $MNAAS_ipvprobe_seq_check_ProcessStatusFilename`.  
   * Confirm Java class works on a sample file:  
     ```bash
     java -cp $CLASSPATHVAR:$MNAAS_Main_JarPath $MNAAS_MNAAS_seqno_check_classname \
          $MNAAS_Property_filename sample_file.txt \
          $MNASS_SeqNo_Check_ipvprobe_history_of_files \
          $MNASS_SeqNo_Check_ipvprobe_missing_history_files \
          $MNASS_SeqNo_Check_ipvprobe_current_missing_files \
          $MNAAS_seq_check_logpath_ipvprobe
     echo $?
     ```  
   * Tail the log while running: `tail -f $MNAAS_seq_check_logpath_ipvprobe`.  

4. **Force a failure for test** – Remove write permission on the status file or set an invalid Java class path; verify that the script creates an SDP ticket and logs the failure.  

---

## 7. External Config / Environment Variables  

| Variable (populated in properties) | Purpose |
|------------------------------------|---------|
| `MNAAS_seq_check_logpath_ipvprobe` | Full path to the script‑specific log file. |
| `MNAAS_ipvprobe_seq_check_ProcessStatusFilename` | Shared status file that stores PID, flags, timestamps, job status, and email‑sent flag. |
| `MNAASMainStagingDirDaily_v1` | Directory where inbound ipvprobe files appear. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Destination directory after successful seq‑check. |
| `FEED_FILE_PATTERN` (associative array) | Mapping *feed_source* → filename glob (e.g., `["IPVPROBE"]="ipvprobe_*.txt"`). |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `MNAAS_MNAAS_seqno_check_classname` | Java runtime configuration for the seq‑no‑check class. |
| `MNAAS_Property_filename` | Path to the generic MNAAS property file passed to the Java class. |
| `MNASS_SeqNo_Check_ipvprobe_current_missing_files` | Temporary file that the Java class populates with missing sequence numbers for the current run. |
| `MNASS_SeqNo_Check_ipvprobe_history_of_files`, `MNASS_SeqNo_Check_ipvprobe_missing_history_files` | Historical tracking files used by the Java class. |
| `ccList`, `GTPMailId` | Email recipients for the missing‑seq alert. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Parameters for the SDP ticket email. |
| `Dname_MNAAS_ipvprobe_seqno_check` | Java process name used for PID detection. |

---

## 8. Suggested Improvements (TODO)

1. **Replace ad‑hoc PID guard with `flock`** – Wrap the entire script body in `flock -n /var/run/MNAAS_ipvprobe_seq_check.lock` to guarantee exclusive execution and automatic lock cleanup on crash.  

2. **Add explicit validation of required properties** – After sourcing the properties file, iterate over a list of mandatory variables, aborting with a clear error if any are empty. This prevents silent failures when a property is misspelled or missing.  

---