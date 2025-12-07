**High‑Level Documentation – `move-mediation-scripts/bin/mnaas_move_files_from_staging_genric.sh`**

---

### 1. Purpose (one‑paragraph summary)

This script orchestrates the ingestion of daily “usage” files that land in the MNAAS staging area. It validates file types, converts non‑CSV Excel files to CSV via a Java utility, checks for duplicate arrivals, optionally validates checksums (or SEM files), moves clean files into the process‑specific data directory, updates a shared *processstatusfile.lst* used by downstream mediation jobs, and raises alerts (email/SDP tickets) when files are missing, duplicated, or fail validation. It is intended to be invoked by a scheduler (e.g., cron) for each logical process (e.g., “IPVProbe”, “Telena”) with parameters that identify the process name and file‑matching pattern.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`init()`** | Loads environment, builds paths (`DupsFilePath`, `datapath`, `logpath`, `processesfilestatus`), redirects `stderr` to the log, and calls `tocsv`. |
| **`tocsv()`** | Scans the landing directory for files matching `$filelike`; for each `.xls`/`.xlsx` file runs the Java class `com.tcl.mnaas.noncsvdataload.FileConverterUtil` to produce a CSV, logs success/failure, and moves the original Excel to the duplicate folder on success. |
| **`FilesDupsCheck()`** | Detects duplicate CSV files by comparing filenames against `primarylist.lst`. Duplicates are moved to the duplicate folder and an alert email is sent; non‑duplicates are logged for processing. |
| **`mv_usage_files_tostaging()`** | Handles the main move logic when files are present: <br>• Updates status flags in `processstatusfile.lst`. <br>• If no files for >24 h, sends a “missing file” alert and creates an SDP ticket. <br>• If files appear after a delay, sends a “recovery” notification. <br>• Performs optional checksum validation (via `mnaas_checksum_validation.sh`) or simple MD5 check; on success moves files to `${datapath}`; on failure would move to a reject folder (code currently commented). <br>• Updates the last‑processed timestamp and resets the delay‑email flag. |
| **`terminateCron_successful_completion()`** | Writes a *Success* status to `processstatusfile.lst`, timestamps the run, logs completion, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion()`** | Logs failure, exits with code 1 (used when conversion or duplicate handling fails). |
| **`email_and_SDP_ticket_triggering_step()`** | Generates a low‑priority SDP ticket (via `mailx`) when the script fails, ensuring only one ticket per run. |
| **`email_and_SDP_ticket_triggering_step_validation()`** | Sends formatted HTML notifications (or SDP tickets) to the Windows team and Hadoop support when Telena files are missing or start arriving after a delay. |
| **Main block** | Parses `getopts` (`-p processname`, `-c checksumMethod`, `-e extension`, `-f filelike`), validates required arguments, calls `init`, checks for an already‑running instance (using PID stored in `processstatusfile.lst`), then branches based on the current flag (`MNAAS_Daily_ProcessStatusFlag`) to run duplicate check, move logic, or IPVProbe‑specific handling. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Command‑line arguments** | `-p <processname>` (mandatory) – logical name of the mediation process.<br>`-c <checksumMethod>` – optional, defaults to `md5sum`.<br>`-e <extension>` – file extension for checksum validation (currently unused).<br>`-f <filelike>` – glob/pattern for files to process (e.g., `*.csv`). |
| **External configuration** | Sourced at start: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop`. This file defines: <br>• `logdir`, `staging_dir`, `mnaasdatalandingdir`, `chesumvalidationlist`, `sourcecodepath`, <br>• email addresses (`SDP_ticket_from_email`, `MOVE_DEV_TEAM`, etc.), <br>• Windows job details (`MNAAS_Telena_feed_windows_*`). |
| **File system side‑effects** | • Creates/updates log file `${logdir}/MNAAS_move_files_from_staging_<process>.log_YYYY-MM-DD`. <br>• Moves original Excel files to `${staging_dir}/${process}/duplicate`. <br>• Moves validated CSV files to `${staging_dir}/${process}/data`. <br>• May move rejected files to `${staging_dir}/${process}/reject` (code presently commented). |
| **Process‑status file** | `${staging_dir}/${process}/config/processstatusfile.lst` – a key‑value file used by many MNAAS scripts. The script reads and writes flags: `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_Daily_ProcessName`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_Feed_FileProcessTime`, `MOVE_Delay_Email_sent`, `MNAAS_Script_Process_Id`. |
| **External services** | • **Java runtime** – runs the conversion JAR (`mnaas.jar` plus many Hadoop/Hive libs). <br>• **Mail** – `mailx` for simple alerts and SDP ticket creation; `/usr/sbin/sendmail` for HTML notifications. <br>• **Logger** – system logger (`logger -s`). |
| **Assumptions** | • All required environment variables are defined in the sourced properties file. <br>• The landing directory contains only files matching the supplied pattern; filenames have no spaces (script uses simple `ls` loops). <br>• The Java converter JAR is compatible with the Hadoop version on the node. <br>• The process‑status file is the single source of truth for downstream scripts. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | How this script interacts |
|---------------------|---------------------------|
| **`mnaas_checksum_validation.sh`** | Invoked (if enabled) to perform checksum validation before moving files. |
| **Downstream mediation jobs** (e.g., `mnaas_generic_recon_loading.sh`, `mnaas_ipvprobe_loading.sh`) | They read the same `processstatusfile.lst` to decide when to start loading data; this script sets the flag to `0` (Success) after moving files. |
| **Scheduler (cron / Oozie)** | Typically called by a cron entry that supplies `-p` and `-f` arguments for each process. |
| **Alerting / ticketing system** | Uses `mailx` to send SDP tickets to `insdp@tatacommunications.com`. |
| **Java conversion utility** (`com.tcl.mnaas.noncsvdataload.FileConverterUtil`) | Performs Excel → CSV conversion; the script only orchestrates its execution. |
| **`processstatusfile.lst`** | Shared status file used by many other MNAAS scripts; this script updates flags, timestamps, and PID. |
| **`primarylist.lst`** | List of expected primary filenames; used for duplicate detection. |
| **`chesumvalidationlist`** | Determines whether checksum validation is required for the current process. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed environment variables** (e.g., `logdir`, `staging_dir`) | Script aborts, files may be left in landing area. | Validate required vars after sourcing the properties file; fail fast with clear error messages. |
| **Concurrent executions** (PID lock may be stale) | Duplicate processing, race conditions on status file. | Implement a lockfile with timeout; verify the stored PID is still alive before skipping. |
| **Java conversion failure** (classpath issues, corrupted Excel) | Files remain in landing area, downstream jobs stall. | Capture Java exit code (already done) and move failing files to a `reject` folder; alert with detailed error. |
| **Duplicate detection false positives** (filename collisions) | Legitimate files moved to duplicate folder, data loss. | Use checksum or file size comparison in addition to name; log both original and duplicate names. |
| **Checksum validation disabled or mis‑configured** | Corrupted data may be loaded downstream. | Enforce checksum validation for all processes via a central config; log the method used. |
| **Email/SDP ticket delivery failure** | Operators unaware of critical issues. | Check mailx/sendmail exit status; retry or fallback to a monitoring system (e.g., Nagios). |
| **Log file growth** | Disk exhaustion on the node. | Rotate logs daily (e.g., via `logrotate`) and limit size. |
| **Filename with spaces or special characters** | `ls`‑based loops break, causing script errors. | Replace back‑ticks loops with `find ... -print0 | while IFS= read -r -d '' file; do ...; done`. |

---

### 6. Typical Execution / Debugging Steps

1. **Prepare environment** – Ensure `mnaaspropertiese.prop` is up‑to‑date and exported.  
2. **Run manually (dry‑run)**:  
   ```bash
   set -x   # enable bash tracing
   ./mnaas_move_files_from_staging_genric.sh -p IPVProbe -c md5 -e csv -f "*.csv"
   ```
3. **Check log** – Tail the generated log file:  
   ```bash
   tail -f ${logdir}/MNAAS_move_files_from_staging_IPVProbe.log_$(date +%F)
   ```
4. **Verify status file** – Confirm flags and timestamps:  
   ```bash
   cat ${staging_dir}/IPVProbe/config/processstatusfile.lst
   ```
5. **Inspect landing & data directories** – Ensure files moved as expected:  
   ```bash
   ls -l ${mnaasdatalandingdir}/
   ls -l ${staging_dir}/IPVProbe/data/
   ```
6. **Validate alerts** – Check mailbox for any SDP ticket or notification generated.  
7. **If failure occurs** – Look for `terminateCron_Unsuccessful_completion` messages in the log; the script will have already moved the problematic file to the duplicate folder.

---

### 7. External Config / Environment Variables Referenced

| Variable | Origin | Usage |
|----------|--------|-------|
| `logdir` | `mnaaspropertiese.prop` | Base directory for script logs. |
| `staging_dir` | `mnaaspropertiese.prop` | Root of per‑process staging area. |
| `mnaasdatalandingdir` | `mnaaspropertiese.prop` | Directory where raw files land before processing. |
| `chesumvalidationlist` | `mnaaspropertiese.prop` | Determines if checksum validation is required for a process. |
| `sourcecodepath` | `mnaaspropertiese.prop` | Path to auxiliary scripts (e.g., `mnaas_checksum_validation.sh`). |
| Email vars (`SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `MOVE_TELENA_WINDOWS_TO_EMAIL`, etc.) | `mnaaspropertiese.prop` | Populated into alert emails and SDP tickets. |
| Windows job vars (`MNAAS_Telena_feed_windows_...`) | `mnaaspropertiese.prop` | Used in validation‑email content. |
| `MNAAS_move_files_from_staging_ScriptName`, `MNAAS_move_files_from_staging_ipvprobe_ScriptName` | Likely defined in the properties file | Used in log messages and ticket generation. |

---

### 8. Suggested Improvements (TODO)

1. **Refactor file‑enumeration loops** – Replace back‑ticks/`ls` with `find … -type f -print0` and a `while read -d ''` loop to safely handle spaces, newlines, and large directories.
2. **Centralize status‑file updates** – Create a small library function (e.g., `set_status key value`) to avoid repetitive `sed -i` commands and reduce the risk of race conditions.

---