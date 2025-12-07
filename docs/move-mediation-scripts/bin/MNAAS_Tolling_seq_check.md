**MNAAS_Tolling_seq_check.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Tolling_seq_check.sh` is a production‑grade Bash orchestration script that validates the sequencing of incoming Tolling data files. For each configured feed source it inspects the staging directory, picks the earliest file, invokes a Java “sequence‑check” utility, and records any missing sequence numbers. Successful runs move the processed file to an “after‑check” directory; failures trigger email alerts and an SDP ticket. The script also maintains a shared process‑status file to coordinate execution with other Move‑Mediation jobs and to prevent concurrent runs.

---

### 2. Key Functions & Responsibilities

| Function / Block | Responsibility |
|------------------|----------------|
| **`seqno_check()`** | Loops over all feed sources defined in `FEED_FILE_PATTERN`. For each source it logs start, counts matching files, selects the oldest file, ensures the Java checker is not already running, invokes the Java class, interprets its exit code, sends missing‑sequence alerts (email + attachment), and moves the file to the post‑check directory. |
| **`terminateCron_successful_completion()`** | Updates the shared process‑status file to indicate success (`MNAAS_job_status=Success`), resets flags, writes run‑time, logs completion, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion()`** | Logs failure, calls `email_and_SDP_ticket_triggering_step`, writes end‑time, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step()`** | Marks the job as `Failure` in the status file, checks whether an alert has already been sent, then sends a notification email to the operational team and creates an SDP ticket via `mailx`. Updates the status file to record that an alert was generated. |
| **Main Program** | Prevents parallel execution by checking the PID stored in the status file, writes the current PID, validates the daily flag (`MNAAS_Daily_ProcessStatusFlag`), invokes `seqno_check`, and routes to the appropriate termination routine. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Tolling_seq_check.properties` – defines all environment variables used (log paths, directories, associative array `FEED_FILE_PATTERN`, Java classpath, JAR path, filenames for history/missing‑file tracking, email lists, SDP ticket parameters, etc.). |
| **External Services** | - **Java class** `MNAAS_seqno_check` (executed via `java -cp …`). <br> - **Mail** (`mail` and `mailx`) for alerts and SDP ticket creation. |
| **File System** | - **Input files**: Tolling feed files matching patterns under `$MNAASMainStagingDirDaily_v1`. <br> - **State files**: `<process‑status‑file>` (shared status), history/missing‑file tracking files (`*_history_of_files`, `*_missing_history_files`, `*_current_missing_files`). <br> - **Output files**: Processed files moved to `$MNAASMainStagingDirDaily_afr_seq_check`. <br> - **Log file**: `$MNAAS_seq_check_logpath_Tolling`. |
| **Side Effects** | - Updates the shared process‑status file (flags, PID, timestamps, job status). <br> - Sends email alerts and SDP tickets on failure or missing sequences. <br> - Moves source files to the “after‑check” directory. |
| **Assumptions** | - All variables referenced are defined in the sourced properties file. <br> - The Java class is present, compatible with the supplied arguments, and returns `0` on success. <br> - The host has `mail`/`mailx` configured and can reach the SMTP server. <br> - The process‑status file is writable by the script user and not concurrently edited by other processes. |

---

### 4. Integration Points (how it connects to other components)

| Connection | Description |
|------------|-------------|
| **Up‑stream** | Other Move‑Mediation ingestion scripts deposit Tolling files into `$MNAASMainStagingDirDaily_v1` using naming conventions that match the patterns defined in `FEED_FILE_PATTERN`. |
| **Shared Status File** | `$MNAAS_Tolling_seq_check_ProcessStatusFilename` is read/written by many Move‑Mediation jobs to coordinate execution order, detect overlapping runs, and expose job status to monitoring dashboards. |
| **Java Sequence‑Check Utility** | The script calls `MNAAS_seqno_check` (located in `$MNAAS_Main_JarPath`). This utility likely updates the history/missing files and may be reused by other feed‑type scripts (e.g., `MNAAS_Sqoop_*`). |
| **Down‑stream** | Files moved to `$MNAASMainStagingDirDaily_afr_seq_check` are consumed by subsequent loading/orchestration scripts (e.g., Sqoop import jobs, Hive loaders). |
| **Alerting / Ticketing** | Email recipients (`${ccList}`, `$GTPMailId`) and SDP ticket endpoint (`insdp@tatacommunications.com`) are shared across the Move‑Mediation suite for incident management. |
| **Cron Scheduler** | The script is intended to be invoked by a cron entry; its PID handling prevents overlapping cron runs. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – stale PID in status file or race condition when two instances start simultaneously. | Duplicate processing, file loss, inconsistent status. | Ensure the status file is atomically locked (e.g., `flock`) before reading/writing PID; add a timeout check for stale PIDs. |
| **Missing or malformed configuration** – undefined variables cause script failure. | Immediate job abort, no alerts. | Validate required variables after sourcing the properties file; fail fast with clear error messages. |
| **Java class failure** – non‑zero exit code not captured correctly. | Files may be moved without proper sequence validation. | Capture Java stdout/stderr to a dedicated log; add retry logic or fallback to a “manual review” flag. |
| **Email/SMTP outage** – alerts not sent. | Operators unaware of missing sequences or failures. | Implement a retry queue or fallback to a log‑only mode; monitor mail queue health. |
| **File system permission issues** – inability to truncate, move, or write logs. | Job stalls, files pile up. | Run a pre‑flight permission check; alert on permission errors via a separate monitoring channel. |
| **Sed in‑place edits on shared status file** – possible corruption if multiple scripts edit simultaneously. | Corrupted status, downstream jobs mis‑interpret flags. | Switch to a more robust key‑value store (e.g., JSON + `jq` or a small SQLite DB) or use file locking around edits. |

---

### 6. Typical Execution & Debugging Steps

1. **Manual Run**  
   ```bash
   cd /app/hadoop_users/MNAAS/
   ./move-mediation-scripts/bin/MNAAS_Tolling_seq_check.sh
   ```
   Ensure the user has read/write access to the properties file, log directory, and status file.

2. **Check Preconditions**  
   - Verify `$MNAAS_Tolling_seq_check_ProcessStatusFilename` exists and is writable.  
   - Confirm `$MNAASMainStagingDirDaily_v1` contains files matching the patterns.  
   - Ensure Java classpath and JAR are accessible.

3. **Observe Logs**  
   - Tail the log: `tail -f $MNAAS_seq_check_logpath_Tolling`.  
   - Look for “START TIME”, “seqno_check … started”, and final “terminated successfully” messages.

4. **Debugging a Failure**  
   - If the script exits with status 1, inspect the log for “terminated Unsuccessfully”.  
   - Check the status file for `MNAAS_job_status=Failure`.  
   - Verify whether an email was sent (`/var/mail/<user>` or mail queue).  
   - Re‑run the Java class manually with the same arguments to capture its output.  

5. **PID / Concurrency Issues**  
   - If you see “Previous MNAAS_seq_check.sh Process is running already”, inspect the PID stored in the status file and confirm the process is still alive: `ps -p <PID>`.  
   - If the PID is stale, clear the entry in the status file or kill the orphaned process.

---

### 7. External Configuration & Environment Variables (from the properties file)

| Variable | Role |
|----------|------|
| `MNAAS_seq_check_logpath_Tolling` | Path to the script’s log file (stderr redirected). |
| `MNAAS_Tolling_seq_check_ProcessStatusFilename` | Shared status file that stores flags, PID, timestamps, etc. |
| `MNAASMainStagingDirDaily_v1` | Directory where incoming Tolling files land. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Destination directory after successful sequence check. |
| `FEED_FILE_PATTERN` (associative array) | Mapping of feed source names → filename glob patterns. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `MNAAS_MNAAS_seqno_check_classname` | Java runtime configuration for the sequence‑check utility. |
| `MNAAS_Property_filename` | Path to the same (or another) properties file passed to the Java class. |
| `MNASS_SeqNo_Check_tolling_history_of_files`, `MNASS_SeqNo_Check_tolling_missing_history_files`, `MNASS_SeqNo_Check_tolling_current_missing_files` | Files used by the Java utility to track processed and missing sequences. |
| `GTPMailId`, `ccList` | Email recipients for alerts. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Parameters used when creating an SDP ticket via `mailx`. |
| `Dname_MNAAS_Tolling_seqno_check` | Name used to identify the Java process (for concurrency guard). |
| `MNAAS_Tolling_seq_check_Scriptname` | Human‑readable script identifier used in logs and alerts. |

*If any of the above variables are missing or empty, the script will likely fail early; a sanity‑check block after sourcing the properties file is recommended.*

---

### 8. Suggested Improvements (TODO)

1. **Add Robust Locking** – Replace the ad‑hoc PID check with a `flock`‑based lock file to guarantee exclusive execution and avoid stale PID problems.  
2. **Externalize Status Management** – Move the key/value status handling from plain‑text `sed` edits to a JSON file managed via `jq` (or a lightweight SQLite DB). This will improve concurrency safety and make downstream parsing easier.  

--- 

*End of documentation.*