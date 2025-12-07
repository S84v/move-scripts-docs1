**File:** `move-mediation-scripts/bin/MNAAS_SimInventory_seq_check.sh`  

---

## 1. Purpose (one‑paragraph summary)

The script validates the sequence‑number integrity of daily SIM‑Inventory files received from two regional sources (Singapore 01 and Holland 01). For each source it picks the earliest pending file, invokes a Java “seq‑no‑check” class to verify that the file follows the expected numeric order, records any gaps in a “missing‑files” report, moves the processed file to an “after‑seq‑check” staging area, and notifies the operations team by e‑mail when gaps are detected. The script also maintains a shared process‑status file that other Move‑Mediation jobs read to coordinate execution order, and it creates an SDP ticket on unrecoverable failures.

---

## 2. Key Functions / Responsibilities

| Function / Block | Responsibility |
|------------------|----------------|
| **`seqno_check_SNG01`** | Process the first pending Singapore 01 SIM‑Inventory file: update status flag, run Java seq‑no validator, log success/failure, generate missing‑file report, e‑mail if gaps, move file to post‑check directory. |
| **`seqno_check_HOL01`** | Same as above but for Holland 01 files (uses flag = 2). |
| **`terminateCron_successful_completion`** | Reset process‑status flags to “idle”, write success metadata (run time, job status), log termination, exit 0. |
| **`terminateCron_Unsuccessful_completion`** | Log failure, invoke ticket‑creation routine, exit 1. |
| **`email_and_SDP_ticket_triggering_step`** | Update status file to “Failure”, send a formatted SDP ticket via `mailx` (if not already sent), log ticket creation. |
| **Main Program** | Guard against concurrent runs, read the shared status flag, decide which regional checks to run, and invoke the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` – defines all `$MNAAS_*` variables (paths, filenames, Java class/JAR, email lists, etc.). |
| **External files** | • Process‑status file (`$MNAAS_SimInventory_seq_check_ProcessStatusFilename`) – read/write flag & metadata.<br>• Log file (`$MNAAS_seq_check_logpath_SimInventory$(date +_%F)`).<br>• Staging directories: `$MNAASMainStagingDirDaily_v1/$Singapore01_SimInventory_extn` and `$.../$Holland01_SimInventory_extn` (input files).<br>• After‑check directory `$MNAASMainStagingDirDaily_afr_seq_check` (output).<br>• History / missing‑files reports (`$MNASS_SeqNo_Check_*`). |
| **Java component** | `$MNAAS_MNAAS_seqno_check_classname` (class) executed with classpath `$CLASSPATHVAR:$MNAAS_Main_JarPath`. Takes the filename and several report files as arguments. |
| **Email** | Uses `mail` (for missing‑file alerts) and `mailx` (for SDP ticket). Recipients are built from `${ccList}`, `$GTPMailId`, `$MOVE_DEV_TEAM`, `$SDP_ticket_from_email`. |
| **System services** | `logger` (writes to syslog), `ps` (process‑count guard), `truncate`, `sed`, `mv`, `ls`, `wc`. |
| **Assumptions** | • All `$MNAAS_*` variables are correctly defined in the properties file.<br>• Java class is present, compatible with the supplied arguments, and returns exit‑code 0 on success.<br>• Mail utilities are configured and can reach external mail servers.<br>• The process‑status file is the single source of truth for downstream jobs. |
| **Side‑effects** | • Updates shared status file (flags, timestamps, job status).<br>• Generates/overwrites missing‑file reports.<br>• Moves processed files to a different staging area.<br>• Sends e‑mail alerts and potentially creates an SDP ticket. |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **`MNAAS_SimInventory_backup_files_process.sh`** (previous step) | Produces the raw SIM‑Inventory files that land in the `$MNAASMainStagingDirDaily_v1/*_SimInventory_extn` directories. |
| **`MNAAS_SimInventory_seq_check.sh`** (this script) | Consumes those files, validates sequence, and moves them to `$MNAASMainStagingDirDaily_afr_seq_check`. |
| **Down‑stream load scripts** (e.g., `MNAAS_SimInventory_load.sh` – not shown) | Read the “after‑seq‑check” directory and rely on the process‑status flag being set to *0* (idle) before starting. |
| **Process‑status file** (`$MNAAS_SimInventory_seq_check_ProcessStatusFilename`) | Shared with all other Move‑Mediation jobs to coordinate execution order and to surface failure information to monitoring dashboards. |
| **Java seq‑no validator** (`$MNAAS_MNAAS_seqno_check_classname`) | Central library used by multiple regional seq‑check scripts (Singapore, Holland, possibly others). |
| **SDP ticketing system** | Triggered via `mailx` to `insdp@tatacommunications.com`; downstream automation consumes the ticket. |
| **Cron scheduler** | The script is intended to be invoked by a daily cron job; the guard `ps aux| grep -w $MNAAS_SimInventory_seq_check_Scriptname` prevents overlapping runs. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Concurrent execution** (multiple instances) | Duplicate processing, race on status file, file loss. | Keep the guard logic but improve it (e.g., lock file in `/var/run`). |
| **Missing/incorrect environment variables** | Script aborts early, status file not updated. | Validate required `$MNAAS_*` variables at start; fail fast with clear log message. |
| **Java validator failure (non‑zero exit)** | Files are not moved; downstream jobs stall. | Capture Java stdout/stderr to a dedicated log; add retry logic or fallback to manual review. |
| **Mail delivery failure** | Operators not notified of missing files or failures. | Check mail command exit status; if non‑zero, write to an “alert” file and raise an alarm via monitoring system. |
| **Unbounded log growth** | Disk exhaustion on log partition. | Rotate logs daily (e.g., via `logrotate`) and compress older files. |
| **Process‑status file corruption** | Downstream jobs misinterpret state. | Write updates atomically (e.g., to a temp file then `mv`), and back up the file before each change. |
| **File system permission changes** | Unable to read/write staging directories. | Run script under a dedicated service account with explicit ACLs; monitor permission drift. |

---

## 6. Typical Execution / Debugging Steps

1. **Manual run (for testing)**  
   ```bash
   export MNAAS_SimInventory_seq_check_Scriptname=MNAAS_SimInventory_seq_check.sh
   ./MNAAS_SimInventory_seq_check.sh
   ```
   *Observe console output (set -x already enabled) and tail the log file*:
   ```bash
   tail -f $MNAAS_seq_check_logpath_SimInventory$(date +_%F)
   ```

2. **Check process‑status flag before and after**  
   ```bash
   grep MNAAS_Daily_ProcessStatusFlag $MNAAS_SimInventory_seq_check_ProcessStatusFilename
   ```

3. **Validate Java component**  
   ```bash
   java -cp $CLASSPATHVAR:$MNAAS_Main_JarPath $MNAAS_MNAAS_seqno_check_classname -h
   ```
   (Run with `-h` or a test file to ensure it starts.)

4. **Inspect missing‑file report**  
   ```bash
   cat $MNASS_SeqNo_Check_siminventory_current_missing_files
   ```

5. **Verify e‑mail alerts**  
   - Check `/var/mail/<user>` or the mail server logs.  
   - Confirm that the attachment file exists and is non‑empty.

6. **Review SDP ticket creation** (if failure)  
   - Search for the ticket in the SDP system using the subject line.  
   - Confirm that the flag `MNAAS_email_sdp_created` is set to `Yes` in the status file.

7. **Cron monitoring**  
   - Ensure the cron entry points to the script with full path and proper environment sourcing.  
   - Verify that the script’s PID does not linger after successful completion.

---

## 7. External Configuration / Environment Variables

| Variable (defined in `MNAAS_ShellScript.properties`) | Role |
|------------------------------------------------------|------|
| `MNAAS_SimInventory_seq_check_ProcessStatusFilename` | Path to the shared status file. |
| `MNAAS_seq_check_logpath_SimInventory` | Base log directory (date‑appended). |
| `MNAASMainStagingDirDaily_v1` | Root of daily staging area. |
| `Singapore01_SimInventory_extn`, `Holland01_SimInventory_extn` | Sub‑directory names for each region. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Destination after seq‑check. |
| `MNASS_SeqNo_Check_siminventory_current_missing_files` | File that holds the list of missing sequence numbers for the current run. |
| `MNASS_SeqNo_Check_siminventory_history_of_files` / `..._missing_history_files` | Historical tracking files used by the Java validator. |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `MNAAS_MNAAS_seqno_check_classname` | Java runtime configuration. |
| `MNAAS_Property_filename` | Path to the same properties file (passed to Java). |
| `Dname_MNAAS_SimInventory_seqno_check` | Name used for the Java process (for ps guard). |
| `ccList`, `GTPMailId`, `MOVE_DEV_TEAM`, `SDP_ticket_from_email` | Email recipients / sender for alerts and tickets. |
| `MNAAS_SimInventory_seq_check_Scriptname` | Script name used in logs and status file. |

*If any of these variables are missing or point to non‑existent paths, the script will fail early; therefore a pre‑flight validation step is advisable.*

---

## 8. Suggested Improvements (TODO)

1. **Introduce a lock‑file mechanism** – replace the `ps aux | grep` guard with an atomic lock file (`flock`) to avoid false positives from grep matching the script name in its own command line.

2. **Add robust error handling around external commands** – capture exit codes of `mail`, `mailx`, `java`, and `mv`; on failure write to a dedicated “error‑alert” file and optionally trigger an immediate SDP ticket, rather than relying on downstream monitoring.

---