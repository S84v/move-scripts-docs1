**MNAAS_Billing_PPU_Geneva.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Billing_PPU_Geneva.sh` is the orchestrator for the daily “Geneva” billing pipeline. It drives the generation of PPU (per‑product‑unit) billing files via a Java JAR, backs up any existing Geneva‑ready files, transfers the newly‑generated files to the external Geneva system via SCP, updates a shared process‑status file, and sends success/failure notifications. The script is intended to run once per day (typically via cron) and includes safeguards against concurrent executions.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_Billing_Geneva_Jar_Execution** | Updates status flag to *1*, logs start, invokes the Java billing JAR (`$GenevaLoaderPPUJarPath`) for the current month, optionally (commented) runs a previous‑month subscription generation on the 1st/2nd of the month, logs success/failure. |
| **MNAAS_Files_Export_To_Geneva** | Sets status flag to *2*, counts files in `$MNAAS_Billing_Geneva_FilesPath`, and if any exist, copies them via `scp` to the remote Geneva server (`geneva@121.244.244.69:/geneva/GENEVACP/import`). |
| **MNAAS_Geneva_Files_Backup** | Sets status flag to *3*, moves all files from `$MNAAS_Billing_Geneva_FilesPath` to the backup directory `$MNAAS_Billing_Geneva_FilesPath_Backup`. |
| **terminateCron_successful_completion** | Resets status flag to *0*, writes success metadata (run time, job status) to the status file, logs completion, emails a success notice, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, records run time, triggers `email_triggering_step`, and exits with code 1. |
| **email_triggering_step** | Marks job status as *Failure* in the status file, checks if an SDP ticket/email has already been created, and if not, sends a failure email and flags the ticket as created. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties` – defines all `$` variables used (paths, log files, JAR location, status file, SCP credentials, etc.). |
| **Environment / Runtime** | Relies on `date`, `sed`, `logger`, `scp`, `mailx`, `java`, `ps`. No explicit environment variables beyond those in the properties file. |
| **Primary Input Files** | - Java property file (`$Mnaas_java_propertypath`).<br>- Existing Geneva billing files in `$MNAAS_Billing_Geneva_FilesPath` (to be backed up or transferred). |
| **Primary Output Files** | - Updated process‑status file (`$MNAASDailyGenevaLoad_ProcessStatusFile`).<br>- Log file (`$GenevaLogPath`).<br>- Backup copies of exported files (`$MNAAS_Billing_Geneva_FilesPath_Backup`). |
| **External Side Effects** | - Executes Java JAR to produce billing files (writes to `$MNAAS_Billing_Geneva_FilesPath`).<br>- Copies files to remote Geneva server via SCP.<br>- Sends email notifications (success & failure). |
| **Assumptions** | - The properties file exists and contains valid, absolute paths.<br>- The remote Geneva host (`121.244.244.69`) is reachable on port 5522 and the `geneva` user has write permission to `/geneva/GENEVACP/import`.<br>- The status file is writable and not corrupted.<br>- The script runs with a user that has read/write/execute rights on all local directories and can run `scp` and `java`. |

---

### 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **MNAAS_Java_Batch.properties** | Centralised configuration used by many MOVE scripts (e.g., other billing/export drivers). |
| **Java JAR (`$GenevaLoaderPPUJarPath`)** | Shared component also invoked by `MNAAS_Billing_Geneva.sh` and other billing scripts. |
| **Process‑status file (`$MNAASDailyGenevaLoad_ProcessStatusFile`)** | Same file is read/written by other daily Geneva‑related scripts (e.g., `MNAAS_Billing_Geneva.sh`, `MNAAS_Actives_tbl_Load.sh`). It stores PID, flags, job status, and email‑ticket flag. |
| **Backup & Export Directories** | Files produced by other MOVE jobs (e.g., usage aggregation) are placed in `$MNAAS_Billing_Geneva_FilesPath`; this script moves them to backup and then exports them. |
| **Mail System** | Uses `mailx` to send notifications; other scripts also use the same address (`raghuram.peddi@contractor.tatatcommunications.com`). |
| **Cron Scheduler** | Typically invoked by a daily cron entry; the script itself checks for an existing PID to avoid overlapping runs. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale PID / false “already running” detection** | Periodically verify that the PID stored in the status file still corresponds to a live process; consider adding a timeout check (e.g., if process > 24 h, clear PID). |
| **Status‑file corruption** | Keep a backup copy of the status file before each write; use atomic `sed` replacements or `mv` a temp file into place. |
| **SCP failure (network, auth, disk space on remote)** | Capture SCP exit code, retry a configurable number of times, and alert on persistent failure. |
| **Java JAR crash or out‑of‑memory** | Ensure Java options (heap size) are defined in the properties file; monitor JVM logs separately. |
| **Missing or malformed configuration** | Add a pre‑flight validation block that checks existence of all required variables and directories; abort early with a clear error. |
| **File permission issues on backup/export directories** | Run a daily permission audit; set appropriate ACLs for the script user. |
| **Email flood on repeated failures** | The script already guards against duplicate SDP tickets; ensure the guard works by testing the flag logic. |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Verify that `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties` exists and contains valid values.  
   - Ensure the executing user can run `java`, `scp`, `mailx`, and write to all paths referenced in the properties file.  

2. **Manual Execution**  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./MNAAS_Billing_PPU_Geneva.sh
   ```
   - The script prints each command (`set -vx`) and logs to `$GenevaLogPath`.  

3. **Debugging Tips**  
   - **Check the status file**: `cat $MNAASDailyGenevaLoad_ProcessStatusFile` to see current flag, PID, and job status.  
   - **Force a clean run**: Edit the status file to set `MNAAS_Daily_ProcessStatusFlag=0` and clear `MNAAS_Script_Process_Id`.  
   - **Increase verbosity**: Add `set -xv` (already present) or insert `echo` statements after each critical command.  
   - **Validate SCP**: Run the SCP command manually with the same source/target to confirm connectivity.  
   - **Inspect Java logs**: The JAR writes to `$GenevaLoaderJavaLogPath`; review for errors.  

4. **Cron Integration**  
   - Typical cron entry (run at 02:00 AM):  
     ```cron
     0 2 * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Billing_PPU_Geneva.sh >> /var/log/mnaas/billing_geneva_cron.log 2>&1
     ```

---

### 7. External Configurations & Variables

| Variable (populated in `MNAAS_Java_Batch.properties`) | Role |
|------------------------------------------------------|------|
| `GenevaLogPath` | Path to the script’s log file. |
| `MNAASDailyGenevaLoad_ProcessStatusFile` | Central status/metadata file shared across Geneva jobs. |
| `MNAASDailyGenevaLoadScriptName` | Human‑readable name of this script (used in logs/emails). |
| `GenevaLoaderPPUJarPath` | Full path to the Java billing JAR. |
| `Mnaas_java_propertypath` | Path to the Java property file passed to the JAR. |
| `GenevaLoaderJavaLogPath` | Log file for the Java process. |
| `MNAAS_Billing_Geneva_FilesPath` | Directory where the JAR writes billing files. |
| `MNAAS_Billing_Geneva_FilesPath_Backup` | Backup directory for exported files. |
| `Geneva_port`, `Geneva_Host`, `Geneva_server`, `Geneva_path` | (Commented out) SCP target details – currently hard‑coded in the script (`-P 5522` and `geneva@121.244.244.69:/geneva/GENEVACP/import`). |
| `Geneva_Files_transfer` | (Commented) Potential wildcard list for files to transfer. |

If any of these variables are missing or empty, the script will fail early; a validation block is recommended.

---

### 8. Suggested Improvements (TODO)

1. **Add Pre‑flight Validation** – Before any processing, verify that all required variables from the properties file are defined, that directories exist and are writable, and that the remote SCP host is reachable. Abort with a clear error and send a failure email if validation fails.

2. **Make SCP Parameters Configurable** – Replace the hard‑coded SCP port (`5522`) and remote host (`121.244.244.69`) with values from the properties file (e.g., `Geneva_SCP_Port`, `Geneva_SCP_Host`). This will align the script with the other Geneva scripts and simplify environment changes.