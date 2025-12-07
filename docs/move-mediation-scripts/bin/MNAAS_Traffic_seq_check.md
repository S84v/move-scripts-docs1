**MNAAS_Traffic_seq_check.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_Traffic_seq_check.sh` is a daily orchestration script that validates the presence and continuity of traffic feed files before they are handed off to the loading pipeline. For each configured feed source it picks the earliest file in the staging area, invokes a Java sequence‑number‑check utility, records success or missing sequence numbers, notifies the operations team via email (and creates an SDP ticket on failure), and moves the processed file to an “after‑check” directory. The script also maintains a shared process‑status file used by downstream load/aggregation jobs to coordinate execution and report overall job health.

---

### 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **seqno_check()** | Loops over all feed sources defined in `FEED_FILE_PATTERN`. For each source it logs start, determines if files exist, runs the Java seq‑no checker, handles the result (missing‑seq email, file move), and updates the process‑status flag. |
| **terminateCron_successful_completion()** | Writes a “Success” state to the process‑status file, logs completion timestamps, and exits with status 0. |
| **terminateCron_Unsuccessful_completion()** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, logs end time, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step()** | Marks the job as “Failure” in the status file, sends a detailed email to the ops team, creates an SDP ticket via `mailx` (if not already created), and updates ticket‑creation flag. |
| **Main Program** (bottom of script) | Prevents concurrent runs by checking the PID stored in the status file, updates the PID, validates the daily flag, calls `seqno_check`, and finally invokes the appropriate termination routine. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Traffic_seq_check.properties` – defines all environment‑specific variables (paths, filenames, email lists, Java classpath, etc.). |
| **External Services** | • Java runtime (`java`) – runs class `$MNAAS_MNAAS_seqno_check_classname`. <br>• Mail system (`mail`, `mailx`) – sends alerts and SDP tickets. <br>• System logger (`logger`). |
| **File System – Inputs** | • Staging directory `$MNAASMainStagingDirDaily_v1` containing feed files matching patterns in associative array `FEED_FILE_PATTERN`. |
| **File System – Outputs** | • Process‑status file `$MNAAS_Traffic_seq_check_ProcessStatusFilename` (flags, timestamps, PID). <br>• Log file `$MNAAS_seq_check_logpath_Traffic`. <br>• “Missing‑seq” report file `$MNASS_SeqNo_Check_traffic_current_missing_files`. <br>• Moved feed files placed in `$MNAASMainStagingDirDaily_afr_seq_check`. |
| **Side Effects** | • Updates shared status file used by downstream load/aggregation scripts. <br>• Sends email alerts and creates SDP tickets on failure. |
| **Assumptions** | • The properties file correctly defines all referenced variables. <br>• Java class is idempotent and returns 0 on success. <br>• Only one instance of the script runs at a time (PID guard). <br>• Mail utilities are configured and reachable from the host. |

---

### 4. Integration with Other Scripts & Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **MNAAS_Traffic_tbl_Load.sh** (and related load scripts) | Reads the same process‑status file; proceeds only when `MNAAS_Daily_ProcessStatusFlag=0` (i.e., after successful seq‑check). |
| **MNAAS_Traffic_Adhoc_Aggr* scripts** | May be triggered later in the same day; rely on the “Success” flag set by this script. |
| **MNAAS_Traffic_seq_check.properties** | Shared across the whole mediation suite; edited by ops when directories, email lists, or Java jar locations change. |
| **Java seq‑no checker JAR** (`$MNAAS_Main_JarPath`) | Central library used by multiple mediation scripts for sequence validation. |
| **SDP ticketing system** (via `mailx`) | Consumes the formatted ticket email; downstream incident‑management processes pick it up. |
| **Cron scheduler** | Typically invokes this script on a daily schedule; the PID guard prevents overlapping runs. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Stale PID in status file** – if the script crashes, the PID may remain, blocking future runs. | Add a sanity check on PID age (e.g., `ps -p $PID -o etime=`) and clear it if older than a threshold. |
| **Missing or malformed properties file** – leads to undefined variables and script failure. | Validate required variables after sourcing; abort with clear error if any are empty. |
| **Java class failure (non‑zero exit)** – currently treated as generic failure. | Capture Java stdout/stderr to a dedicated log and include it in the failure email for faster root‑cause analysis. |
| **Mail delivery failure** – alerts or SDP tickets not sent. | Check mail command exit status; retry or fallback to a secondary notification channel (e.g., Slack webhook). |
| **File loss during move** – if `mv` fails, files may stay in staging and be reprocessed. | Verify the move succeeded (`[ -e "$dest/$filename" ]`) and log an error if not. |
| **Concurrent execution** – race condition if two cron entries overlap. | Keep the PID guard but also lock a filesystem lock file (`flock`) for extra safety. |

---

### 6. Running / Debugging the Script  

1. **Standard execution (cron)**  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Traffic_seq_check.sh
   ```
   The script logs verbosely (`set -x`) to the configured log file.

2. **Manual run (debug mode)**  
   ```bash
   export MNAAS_Traffic_seq_check_properties=/path/to/MNAAS_Traffic_seq_check.properties
   . $MNAAS_Traffic_seq_check_properties
   bash -x MNAAS_Traffic_seq_check.sh   # shows each command as it executes
   ```

3. **Checking status**  
   ```bash
   cat $MNAAS_Traffic_seq_check_ProcessStatusFilename
   ```
   Look for `MNAAS_job_status=Success` or `Failure` and the PID.

4. **Force re‑run** (if PID is stale)  
   ```bash
   sed -i 's/^MNAAS_Script_Process_Id=.*/MNAAS_Script_Process_Id=0/' $MNAAS_Traffic_seq_check_ProcessStatusFilename
   ```

5. **Inspect Java output**  
   The Java class writes to stdout/stderr; redirect if needed:
   ```bash
   java ... 2>&1 | tee -a $MNAAS_seq_check_logpath_Traffic
   ```

6. **Verify email/SDP ticket**  
   Check the mail logs (`/var/log/maillog` or equivalent) for the alert and ticket messages.

---

### 7. External Configuration / Environment Variables  

| Variable (defined in properties) | Role |
|----------------------------------|------|
| `MNAAS_seq_check_logpath_Traffic` | Path to the script’s log file. |
| `MNAAS_Traffic_seq_check_ProcessStatusFilename` | Shared status file updated by many mediation scripts. |
| `MNAASMainStagingDirDaily_v1` | Directory where raw traffic feed files land. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Destination for files after sequence check. |
| `FEED_FILE_PATTERN` (associative array) | Mapping of feed source → filename glob pattern. |
| `MNAAS_Main_JarPath` / `MNAAS_MNAAS_seqno_check_classname` | Java JAR location and main class for seq‑no validation. |
| `MNASS_SeqNo_Check_traffic_current_missing_files` | Temp file that holds list of missing sequence numbers. |
| `GTPMailId`, `ccList`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / headers for alerts and ticket creation. |
| `Dname_MNAAS_Traffic_seqno_check` | Java process name used for PID guard. |
| `MNAAS_FlagValue` (derived) | Controls whether the script proceeds (0 or 1). |

If any of these variables are missing or point to non‑existent paths, the script will fail early; therefore the properties file must be validated during deployment.

---

### 8. Suggested TODO / Improvements  

1. **Add explicit configuration validation** – after sourcing the properties file, loop through a required‑variable list and abort with a clear message if any are empty or point to invalid locations.  
2. **Replace ad‑hoc PID guard with `flock`** – acquire an exclusive lock file at script start (`exec 200>/var/lock/MNAAS_Traffic_seq_check.lock && flock -n 200 || exit 0`) to guarantee single‑instance execution even if the status file becomes corrupted.  

--- 

*End of documentation.*