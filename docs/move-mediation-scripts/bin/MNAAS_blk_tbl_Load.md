**File:** `move-mediation-scripts/bin/MNAAS_blk_tbl_Load.sh`  

---

## 1. High‑Level Summary
`MNAAS_blk_tbl_Load.sh` orchestrates the creation and rename of the **Traffic Details block table** used in the Move‑Mediation data‑pipeline. It drives two Java jobs (insert‑into‑block‑table and rename‑block‑table) via a configurable JAR, updates a shared *process‑status* file, writes detailed logs, and on failure generates an SDP ticket and notification email. The script is intended to be run from a nightly/interval cron, guarded against concurrent executions, and integrates with the broader Move‑Mediation ecosystem (e.g., other block‑table scripts, sequence‑check scripts, Impala refreshes).

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **Traffic_Details_blockconsolidation_creation** | Sets status flag = 1, launches the *Insert_Blk_table* Java class to load raw traffic data into a temporary block table, logs success/failure, and aborts on error. |
| **Traffic_Details_blockconsolidation_rename** | Sets status flag = 2, launches the *Rename_Blk_table* Java class to replace the production block table with the newly loaded one, logs success/failure, and aborts on error. |
| **terminateCron_successful_completion** | Resets status flag = 0, marks job status = Success, clears email‑ticket flag, writes final timestamps to the status file, logs completion, and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes **email_and_SDP_ticket_triggering_step**, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | Marks job status = Failure, checks if an SDP ticket/email has already been raised, and if not sends a templated email via `mailx` to the SDP ticketing address, updates the status file to indicate ticket creation, and logs the action. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`. Expected variables (examples):<br>• `MNAAS_Blk_ProcessStatusFileName` – path to the shared status file.<br>• `MNAAS_Blk_TrafficDetails_CntrlFileName` – control file for the block table.<br>• `Blockconsolidation_JarName` – JAR containing the Java classes.<br>• `Insert_Blk_table`, `Rename_Blk_table` – fully‑qualified Java class names.<br>• `MNAAS_BlkTrafficDetailsLogPath` – directory for daily log files.<br>• `MNAASblkScriptName` – script identifier used for process checks.<br>• Email‑related vars: `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `ccList`, `GTPMailId`. |
| **External Services** | • **Java runtime** (calls to `java -cp …`)<br>• **Impala** – optional refresh command (currently commented out).<br>• **mailx** – sends SDP ticket email.<br>• **syslog/logger** – writes to system log and script‑specific log files.<br>• **File system** – reads/writes status, control, and log files. |
| **Primary Input Data** | Traffic detail source files (location defined inside the control file referenced by `$MNAAS_Blk_TrafficDetails_CntrlFileName`). |
| **Outputs** | • Temporary block table (created by Java insert job).<br>• Production block table (replaced by rename job).<br>• Updated status file (`MNAAS_Blk_ProcessStatusFileName`).<br>• Daily log file (`$MNAAS_BlkTrafficDetailsLogPath$(date +_%F)`).<br>• Optional SDP ticket email (to `insdp@tatacommunications.com`). |
| **Side Effects** | • May trigger Impala table refresh (if uncommented).<br>• Alters shared status file used by other Move‑Mediation scripts (e.g., sequence‑check, other block‑table loaders). |
| **Assumptions** | • Only one instance of this script runs at a time (guarded by `ps` checks).<br>• Java classes are idempotent and can be safely re‑run if needed.<br>• The status file exists and is writable by the script user.<br>• Mail server is reachable and `mailx` is configured correctly.<br>• All environment variables defined in the properties file are valid. |

---

## 4. Interaction with Other Components  

| Component | How `MNAAS_blk_tbl_Load.sh` Connects |
|-----------|--------------------------------------|
| **MNAAS_ShellScript.properties** | Centralised configuration source for all Move‑Mediation scripts; any change propagates to this script. |
| **Other block‑table loaders** (`MNAAS_Traffic_tbl_with_nodups_loading.sh`, `MNAAS_Traffic_seq_check.sh`, etc.) | Share the same status file (`MNAAS_Blk_ProcessStatusFileName`). The flag values (0, 1, 2) dictate whether those scripts should start or wait. |
| **Sequence‑check script** (`MNAAS_seq_check_Scriptname` placeholder) | Reads the same status file to verify that the block‑table load succeeded before downstream processing. |
| **Impala / Hive** | The script contains commented‑out Impala refresh commands; downstream analytics jobs depend on the refreshed tables. |
| **SDP ticketing system** | Failure notification is sent via email to `insdp@tatacommunications.com`; the ticketing system parses the email to create an incident. |
| **Cron scheduler** | Typically invoked by a nightly cron entry; the script’s own process‑count guard prevents overlapping runs. |
| **Java JAR (`Blockconsolidation_JarName`)** | Provides the `Insert_Blk_table` and `Rename_Blk_table` classes that perform the heavy data‑movement logic. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (multiple script instances) | Duplicate loads, table contention, status file corruption | Keep the `ps` guard; add a lock file (`flock`) for extra safety. |
| **Stale status flag** (e.g., flag = 1 left after a crash) | Subsequent runs skip necessary steps | Implement a timeout check on the flag; reset to 0 if older than a configurable threshold. |
| **Java job failure not captured** (non‑zero exit but script continues) | Incomplete data load, downstream jobs see partial data | Ensure Java classes exit with proper status; add explicit error‑code checks after each `java` call (already present). |
| **Mail/SPI ticket not sent** (mailx mis‑config) | Failure goes unnoticed, no incident raised | Add a fallback log entry and optionally a secondary alert (e.g., Slack webhook). |
| **Log file growth** (daily logs never rotated) | Disk exhaustion | Configure logrotate for `$MNAAS_BlkTrafficDetailsLogPath*`. |
| **Missing/invalid properties** | Script aborts early, unclear error | Validate required variables after sourcing the properties file; exit with clear message if any are empty. |
| **Impala refresh omitted** | Downstream analytics read stale data | Review whether the refresh should be re‑enabled; if so, add error handling around the `impala-shell` command. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the user has read/write access to the status file, control file, and log directory.  
   - Verify Java is installed and `$CLASSPATHVAR` and `$Blockconsolidation_JarName` point to a valid JAR.  
   - Confirm `mailx` can send mail to `insdp@tatacommunications.com`.  

2. **Typical Invocation** (via cron)  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties && \
   /path/to/MNAAS_blk_tbl_Load.sh
   ```  

3. **Manual Run (debug mode)**  
   ```bash
   set -x   # already present, prints each command
   ./MNAAS_blk_tbl_Load.sh
   ```  
   - Watch the console for the `ps` guard output.  
   - After completion, inspect the daily log file:  
     ```bash
     tail -f $MNAAS_BlkTrafficDetailsLogPath$(date +_%F)
     ```  

4. **Checking Process Status**  
   ```bash
   grep MNAAS_blk_ProcessStatusFlag $MNAAS_Blk_ProcessStatusFileName
   ```  

5. **Force Reset (if stuck)**  
   ```bash
   sed -i 's/MNAAS_blk_ProcessStatusFlag=.*/MNAAS_blk_ProcessStatusFlag=0/' $MNAAS_Blk_ProcessStatusFileName
   ```  

6. **Common Debug Points**  
   - Verify the Java class names (`$Insert_Blk_table`, `$Rename_Blk_table`) are correct.  
   - Ensure the control file (`$MNAAS_Blk_TrafficDetails_CntrlFileName`) lists the source data files in the expected format.  
   - If the script aborts in `terminateCron_Unsuccessful_completion`, check the email log and SDP ticket creation.  

---

## 7. External Configurations & Environment Variables  

| Variable (from properties) | Purpose | Typical Example |
|----------------------------|---------|-----------------|
| `MNAAS_Blk_ProcessStatusFileName` | Shared status file for block‑table processes. | `/app/hadoop_users/MNAAS/status/blk_process_status.txt` |
| `MNAAS_Blk_TrafficDetails_CntrlFileName` | Control file listing source files & parameters for the block load. | `/app/hadoop_users/MNAAS/ctrl/traffic_details.ctrl` |
| `Blockconsolidation_JarName` | JAR containing the Java loader/renamer classes. | `blockconsolidation.jar` |
| `Insert_Blk_table` / `Rename_Blk_table` | Fully‑qualified Java class names invoked by the script. | `com.tata.move.InsertBlkTable` |
| `MNAAS_BlkTrafficDetailsLogPath` | Directory for daily log files. | `/app/hadoop_users/MNAAS/logs/blk_traffic/` |
| `MNAASblkScriptName` | Identifier used for process‑count guard. | `MNAAS_blk_tbl_Load.sh` |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `ccList`, `GTPMailId` | Email routing for failure notifications. | `move-support@tatacommunications.com` |
| `IMPALAD_HOST` (optional) | Host for Impala refresh commands (currently commented). | `impala01.tata.com` |

*If any of these variables are missing or empty, the script will likely fail early; a sanity‑check block after sourcing the properties file is recommended.*

---

## 8. Suggested Improvements (TODO)

1. **Add a lock‑file mechanism** (`flock` or PID file) to guarantee single‑instance execution even if the `ps` guard is fooled by similarly named processes.  
2. **Validate required properties** immediately after sourcing the properties file and abort with a clear error message if any are undefined or point to non‑existent files/JARs.  

--- 

*End of documentation.*