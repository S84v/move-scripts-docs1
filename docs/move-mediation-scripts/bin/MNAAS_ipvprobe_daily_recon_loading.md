**File:** `move-mediation-scripts/bin/MNAAS_ipvprobe_daily_recon_loading.sh`

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily “IPVProbe” reconciliation workflow. It moves inbound probe usage files from the staging area to an intermediate directory, loads them into a temporary Hive table via a Java loader, merges the data into the final Hive/Impala table (partitioned by file date), backs the raw files up to a daily backup directory, and records progress in a shared *process‑status* file. The script is guarded against concurrent execution, writes detailed logs, and on failure raises an SDP ticket and sends an email notification.

---

## 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **`MNAAS_ipvprobe_daily_recon_enrich`** | Moves daily probe files (`$IPVPROBE_Daily_Recon_ext`) from the main staging directory to `$MNAASInterFilePath_Daily_IPVProbe_Recon_Details/`; strips any existing `File_Name` header line. |
| **`MNAAS_ipvprobe_daily_recon_temp_table_load`** | Clears target HDFS path, copies the intermediate files to HDFS, invokes the Java class `$Load_nonpart_table` to bulk‑load them into a *temporary* Hive table (`$ipvprobe_daily_recon_inter_tblname`). Refreshes the table in Impala. |
| **`MNAAS_ipvprobe_daily_recon_loading`** | Drops existing partitions in the final table (`$ipvprobe_daily_recon_tblname`) for the dates present in the temp table, then inserts the temp data into the final table (dynamic partitioning). Refreshes the Impala metadata. |
| **`MNAAS_cp_daliy_recon_files_to_backup`** | Copies the raw files from the intermediate directory to the daily backup directory (`$Daily_IPVProbe_BackupDir`). |
| **`terminateCron_successful_completion`** | Writes a “Success” status (flag = 0) to the process‑status file, logs completion, and exits 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, updates status, invokes `email_and_SDP_ticket_triggering_step`, and exits 1. |
| **`email_and_SDP_ticket_triggering_step`** | Sends a failure email to `$GTPMailId` (CC `$ccList`) and marks the SDP‑ticket‑created flag to avoid duplicate tickets. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_ipvprobe_daily_recon_loading.properties` – defines all environment variables used (paths, DB names, table names, Hadoop/Impala hosts, email lists, log file name, etc.). |
| **External Services** | - Hadoop HDFS (`hadoop fs` commands) <br> - Hive (`hive -e`) <br> - Impala (`impala-shell`) <br> - Java loader JAR (`$Load_nonpart_table`) <br> - System logger (`logger`) <br> - Mail subsystem (`mail`) |
| **File System** | - Reads inbound files from `$MNAASMainStagingDirDaily/$IPVPROBE_Daily_Recon_ext` <br> - Writes intermediate files to `$MNAASInterFilePath_Daily_IPVProbe_Recon_Details/` <br> - Copies to HDFS `$MNAAS_Daily_Recon_load_ipvprobe_PathName/` <br> - Backs up raw files to `$Daily_IPVProbe_BackupDir/` <br> - Updates process‑status file `$MNAAS_ipvprobe_Daily_Recon_ProcessStatusFileName` |
| **Process‑Status File** | Holds flags (`MNAAS_Daily_ProcessStatusFlag`), current step name, PID, job status, timestamps, and email‑ticket flag. Used for restartability and concurrency guard. |
| **Outputs** | - Populated Hive tables (`$ipvprobe_daily_recon_inter_tblname` and `$ipvprobe_daily_recon_tblname`) <br> - Log file `$MNAAS_ipvprobe_daily_recon_loadingLogName` (stderr redirected) <br> - Backup copies of raw files <br> - Email & SDP ticket on failure |
| **Assumptions** | - All required environment variables are defined in the properties file. <br> - Hadoop, Hive, Impala services are reachable and the executing user has required permissions. <br> - Java loader JAR and its dependencies are present on the classpath. <br> - Mail service (`mail`) is configured on the host. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | How it Links |
|---------------------|--------------|
| **Other “MNAAS_*_daily_recon_*” scripts** | Follow the same process‑status file convention; the flag values (0‑4) allow a later script to resume from the last successful step if this script crashes. |
| **Edge‑node backup orchestration** (`MNAAS_edgenode*_backup_files_process*.sh`) | The backup directory `$Daily_IPVProbe_BackupDir` is later consumed by nightly backup jobs that archive raw CDR files. |
| **SDP ticketing system** | Triggered via `email_and_SDP_ticket_triggering_step`; the email body references the script name, and downstream automation parses the email to create an SDP ticket. |
| **Hive/Impala tables** | Final tables (`$ipvprobe_daily_recon_tblname`) are read by downstream analytics pipelines (e.g., billing, reporting). |
| **Java loader** (`Load_nonpart_table`) | Shared across many daily‑load scripts; expects the same HDFS input layout and table schema. |
| **Cron scheduler** | The script is intended to be launched by a daily cron entry; the PID guard prevents overlapping runs. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Missing or malformed inbound files** | Add pre‑flight validation (e.g., file count > 0, checksum) before moving files; log and abort early if validation fails. |
| **Hadoop/Hive/Impala service outage** | Implement retry loops with exponential back‑off for `hadoop fs`, `hive`, and `impala-shell` commands; alert on repeated failures. |
| **Java loader failure (non‑zero exit)** | Capture stdout/stderr of the Java process to a dedicated log; consider a fallback to a Spark loader if persistent. |
| **Concurrent execution race** | The PID guard is simple; enhance by using a lock file (`flock`) to guarantee atomic acquisition. |
| **Permission issues on directories or HDFS paths** | Verify directory permissions at script start; exit with clear error if not writable/readable. |
| **Email delivery failure** | Check mail command exit status; if failed, write to a local “failed‑email” queue for later retry. |
| **Process‑status file corruption** | Keep a backup copy of the status file before each write; use atomic `sed -i` replacements or `mv` a temp file. |

---

## 6. Running / Debugging the Script  

1. **Manual Invocation**  
   ```bash
   export MNAAS_IPVPROBE_DAILY_RECON_SCRIPTNAME=MNAAS_ipvprobe_daily_recon_loading.sh   # optional override
   ./MNAAS_ipvprobe_daily_recon_loading.sh
   ```
   Ensure the properties file is present at the hard‑coded path or set `MNAAS_Property_File` before sourcing.

2. **Check Log**  
   The script redirects `stderr` to `$MNAAS_ipvprobe_daily_recon_loadingLogName`. Tail the log while running:
   ```bash
   tail -f "$MNAAS_ipvprobe_daily_recon_loadingLogName"
   ```

3. **Force a Specific Step**  
   Edit the process‑status file (`$MNAAS_ipvprobe_Daily_Recon_ProcessStatusFileName`) and set `MNAAS_Daily_ProcessStatusFlag` to the desired step number (1‑4). The main block will resume from that step.

4. **Debug Mode**  
   The script already runs with `set -x` (trace). For deeper inspection, add `set -e` to abort on any command failure, or insert `echo` statements before critical commands.

5. **Verify PID Guard**  
   If you see “Previous … Process is running already”, check the PID stored in the status file:
   ```bash
   grep MNAAS_Script_Process_Id "$MNAAS_ipvprobe_Daily_Recon_ProcessStatusFileName"
   ps -p <PID> -o cmd=
   ```

6. **Simulate Failure**  
   Force a non‑zero exit from the Java loader (e.g., rename the JAR) to test the email/SDP ticket path.

---

## 7. External Configuration & Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_ipvprobe_daily_recon_loadingLogName` | Path of the script’s log file (stderr). |
| `MNAAS_ipvprobe_Daily_Recon_ProcessStatusFileName` | Central status file tracking flags, PID, timestamps, etc. |
| `MNAASMainStagingDirDaily` | Root staging directory for inbound files. |
| `IPVPROBE_Daily_Recon_ext` | Filename pattern (extension) of daily recon files. |
| `MNAASInterFilePath_Daily_IPVProbe_Recon_Details` | Intermediate directory where files are moved before HDFS copy. |
| `MNAAS_Daily_Recon_load_ipvprobe_PathName` | HDFS target directory for the bulk load. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `Load_nonpart_table` | Java classpath and loader JAR details. |
| `nameNode`, `dbname`, `ipvprobe_daily_recon_inter_tblname`, `ipvprobe_daily_recon_tblname` | Hive database and table identifiers. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive and Impala. |
| `Daily_IPVProbe_BackupDir` | Filesystem backup location for raw probe files. |
| `GTPMailId`, `ccList` | Email recipients for failure notifications. |
| `MNAAS_Sqoop_ba_rep_data_Scriptname` | Used only for logging; likely a generic script name placeholder. |

If any of these variables are missing or empty, the script will fail at runtime; therefore the properties file must be validated before deployment.

---

## 8. Suggested Improvements (TODO)

1. **Replace ad‑hoc PID guard with `flock`** – ensures atomic lock acquisition and automatic release on script exit, reducing race‑condition risk.  
2. **Externalize command exit‑code handling** – wrap critical commands (`hadoop fs`, `hive`, `impala-shell`, Java loader) in a helper function that logs, retries, and aborts cleanly, improving maintainability and observability.  

---