**High‑Level Documentation – `move-mediation-scripts/bin/mnaas_seq_check_for_feed.sh`**

---

### 1. Purpose (one‑paragraph summary)

`mnaas_seq_check_for_feed.sh` is a production‑grade Bash driver that validates the sequence integrity of incoming feed files for a given MNAAS (Mobile Network Analytics & Assurance System) process. It inspects the staging area for files matching a supplied pattern, invokes a Java‑based `SequanceValidation` routine, updates a shared process‑status file, logs activity, and raises alerts (email + SDP ticket) when missing or out‑of‑order files are detected. The script also enforces single‑instance execution via a lock file and reports success or failure back to the orchestration framework.

---

### 2. Key Functions / Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`init`** | Builds the log file path, resolves the status‑file location, sets the data directory, and redirects `stderr` to the log. |
| **`seqno_check`** | *Core logic*: <br>• Marks the status file as “running”. <br>• Counts matching files; if none, exits quietly. <br>• Picks the oldest matching file, creates a lock, and runs the Java validator `com.tcl.mnass.validation.SequanceValidation`. <br>• On success, checks the *missing‑files* list and sends a notification email if any are present. |
| **`terminateCron_successful_completion`** | Resets status flags to “idle/success”, records run time, removes the lock, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, removes the lock, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates the status file to “Failure”, ensures an SDP ticket/email is sent only once per run, and logs the action. |
| **Main program** | Parses CLI options (`-p` process name, `-e` file extension – currently unused, `-f` file pattern), validates required arguments, calls `init`, checks for an existing lock, writes the current PID to the status file, decides whether to run `seqno_check` based on the daily flag, and finally calls the appropriate termination routine. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Command‑line inputs** | `-p <processname>` (mandatory), `-e <fileextension>` (optional, not used), `-f <filelike>` (glob/pattern for files to validate). |
| **External configuration** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop` – defines `logdir`, `staging_dir`, `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, etc. |
| **File system inputs** | <ul><li>`$staging_dir/${processname}/data/${filelike}` – feed files to be validated.</li><li>`$staging_dir/${processname}/config/seqcheckproccessstatusfile.lst` – shared status file.</li><li>`$staging_dir/${processname}/config/*.lst` – various supporting lists (missingfiles, processedhistory, etc.).</li></ul> |
| **File system outputs** | <ul><li>Log file: `$logdir/mnaas_move_files_from_seqcheck_${processname}.log_YYYY-MM-DD`.</li><li>Lock file: `$staging_dir/${processname}/lock.lck` (created/removed).</li><li>Potentially moved/renamed feed file (commented out in script).</li></ul> |
| **Side effects** | <ul><li>Updates status file fields (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, timestamps, etc.).</li><li>Sends email alerts (standard `mail` and `mailx` for SDP ticket).</li><li>Invokes Java validator JAR (requires Java runtime, correct classpath, and the `MNAAS_Main_JarPath`).</li></ul> |
| **Assumptions** | <ul><li>All directories (`logdir`, `staging_dir`) exist and are writable by the script user.</li><li>Java class `com.tcl.mnass.validation.SequanceValidation` is present in `$MNAAS_Main_JarPath` and compatible with the supplied arguments.</li><li>Mail utilities (`mail`, `mailx`) are configured and can reach internal distribution lists.</li><li>Only one instance per process runs at a time (enforced by lock file).</li></ul> |

---

### 4. Integration Points (how it connects to other components)

| Connection | Description |
|------------|-------------|
| **Status file (`seqcheckproccessstatusfile.lst`)** | Shared with other MNAAS scripts (e.g., `mnaas_move_files_from_seqcheck_*.sh`, `mnaas_generic_recon_loading.sh`). Those scripts read the same flags to decide whether to start downstream loads. |
| **Staging area (`$staging_dir/${processname}/data`)** | Populated by upstream ingestion scripts (e.g., SFTP pullers, Hadoop move scripts). This script validates the order before downstream processing. |
| **Java validator** | The `SequanceValidation` class may also be invoked by other validation scripts (e.g., `mnaas_checksum_validation.sh`). |
| **Alerting / ticketing** | Uses internal mailing lists (`ccList`, `MOVE_DEV_TEAM`) and SDP ticketing (`insdp@tatacommunications.com`). Other monitoring tools (e.g., Nagios, Splunk) may ingest the log entries. |
| **Lock file** | Prevents concurrent runs; other scripts check the same lock path to avoid clashes. |
| **Environment properties** | The property file is common across the MNAAS suite; any change propagates to all scripts that source it. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale lock file** (e.g., script crashes, lock not removed) | Subsequent runs are skipped → data backlog | Add a watchdog that removes lock files older than a configurable threshold; log lock‑file age on start. |
| **Incorrect status‑file updates** (sed in‑place may corrupt file if interrupted) | Downstream processes may mis‑interpret flags | Use atomic file updates (write to temp file then `mv`) or `ed` with backup. |
| **Java validator failure** (non‑zero exit, memory OOM) | Missing alerts, possible data loss | Capture Java stdout/stderr to a dedicated log; set a timeout (`timeout` command) and monitor exit codes. |
| **Mail delivery failure** (SMTP down) | No alert sent, issue goes unnoticed | Implement retry logic; fallback to writing an alert file that a monitoring daemon can pick up. |
| **Pattern (`filelike`) mismatch** leading to zero files processed | Silent success, but data not validated | Log the pattern used and the file count; raise a warning if count = 0 for more than N consecutive runs. |
| **Hard‑coded paths** (e.g., property file location) | Breaks when environment changes | Parameterise the property file path via an env variable or CLI flag. |

---

### 6. Typical Execution & Debugging Steps

1. **Run the script** (example for process *IPVPROBE* and files ending `.csv`):  

   ```bash
   ./mnaas_seq_check_for_feed.sh -p IPVPROBE -f "*.csv"
   ```

2. **Check the log** (path printed by `init`):  

   ```bash
   tail -f /var/log/mnaas/mnaas_move_files_from_seqcheck_IPVPROBE.log_2025-12-04
   ```

3. **Verify lock handling** (if script exits early):  

   ```bash
   ls -l $staging_dir/IPVPROBE/lock.lck
   ```

4. **Inspect status file** for flag values:  

   ```bash
   cat $staging_dir/IPVPROBE/config/seqcheckproccessstatusfile.lst
   ```

5. **Debug Java call** – add `-Xmx1g` or other JVM options in `CLASSPATHVAR` if needed; capture its output:  

   ```bash
   java -Dname="MNAAS_IPVPROBE_seqno_check" -cp $CLASSPATHVAR:$MNAAS_Main_JarPath \
        com.tcl.mnass.validation.SequanceValidation ... 2>&1 | tee /tmp/java_debug.log
   ```

6. **Force a clean start** (remove stale lock & reset flag):  

   ```bash
   rm -f $staging_dir/IPVPROBE/lock.lck
   sed -i 's/MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=0/' \
       $staging_dir/IPVPROBE/config/seqcheckproccessstatusfile.lst
   ```

---

### 7. External Configuration / Environment Variables Used

| Variable | Source | Meaning |
|----------|--------|---------|
| `logdir` | `mnaaspropertiese.prop` | Base directory for script logs. |
| `staging_dir` | `mnaaspropertiese.prop` | Root of per‑process staging area. |
| `CLASSPATHVAR` | `mnaaspropertiese.prop` | Additional Java classpath entries. |
| `MNAAS_Main_JarPath` | `mnaaspropertiese.prop` | Path to the JAR containing `SequanceValidation`. |
| `ccList` | `mnaaspropertiese.prop` | Comma‑separated list of CC recipients for alerts. |
| `GTPMailId` | `mnaaspropertiese.prop` | Primary recipient for missing‑file alerts. |
| `SDP_ticket_from_email` | `mnaaspropertiese.prop` | Sender address for SDP ticket emails. |
| `MOVE_DEV_TEAM` | `mnaaspropertiese.prop` | CC list for SDP tickets. |
| `processname`, `filelike` | CLI arguments (`-p`, `-f`) | Determines which process and file pattern to act on. |

---

### 8. Suggested Improvements (TODO)

1. **Robust lock & status handling** – replace in‑place `sed` edits with atomic writes (e.g., `mv` a temp file) and add lock‑file age validation to avoid dead‑locks.
2. **Parameter validation & defaults** – enforce that `filelike` is non‑empty, validate that required directories exist before proceeding, and expose the property‑file path as a CLI option for easier testing across environments.  

---