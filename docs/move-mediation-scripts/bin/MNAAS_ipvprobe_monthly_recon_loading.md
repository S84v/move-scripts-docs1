**File:** `move-mediation-scripts/bin/MNAAS_ipvprobe_monthly_recon_loading.sh`

---

## 1. High‑Level Summary
This Bash driver orchestrates the monthly “IPVProbe” reconciliation data‑load pipeline. It moves raw monthly recon files from the daily staging area to an intermediate directory, strips header rows, loads the data into a temporary Hive table (via Hadoop FS copy and a Java “Load_nonpart_table” jar), refreshes the table in Impala, then merges the data into the final partitioned Hive table. After a successful load the raw files are copied to a backup directory. Throughout the run the script updates a shared *process‑status* file, writes detailed logs, and on failure raises an SDP ticket and email notification. The script is intended to be executed by a nightly/monthly cron job and includes a PID‑guard to avoid concurrent executions.

---

## 2. Important Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `MNAAS_ipvprobe_monthly_recon_enrich` | Detects raw monthly recon files (`$IPVPROBE_Monthly_Recon_ext`) in `$MNAASMainStagingDirDaily`, moves each to `$MNAASInterFilePath_Monthly_IPVProbe_Recon_Details/`, removes the first line (header), and logs activity. |
| `MNAAS_ipvprobe_monthly_recon_temp_table_load` | Copies the enriched files to HDFS (`$MNAAS_Monthly_Recon_load_ipvprobe_PathName`), runs the Java loader (`$Load_nonpart_table`) to populate a temporary Hive table (`$ipvprobe_monthly_recon_inter_tblname`), refreshes the table in Impala, and handles process‑already‑running detection. |
| `MNAAS_ipvprobe_monthly_recon_loading` | Drops existing partitions for the months present in the temp table, then inserts the temp data into the final Hive table (`$ipvprobe_monthly_recon_tblname`) using dynamic partitioning, refreshes the Impala view, and logs success/failure. |
| `MNAAS_cp_monthly_recon_files_to_backup` | Copies the raw files from the intermediate directory to the daily backup directory (`$Daily_IPVProbe_BackupDir`). |
| `terminateCron_successful_completion` | Resets the process‑status flag to *idle* (`0`), marks job status *Success*, timestamps the run, logs completion, and exits `0`. |
| `terminateCron_Unsuccessful_completion` | Logs failure, timestamps the run, invokes `email_and_SDP_ticket_triggering_step`, and exits `1`. |
| `email_and_SDP_ticket_triggering_step` | Sets job status *Failure*, checks the “email‑SDP‑created” flag, sends a pre‑formatted email to `$GTPMailId` (CC `$ccList`) and marks the flag to avoid duplicate tickets. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_ipvprobe_monthly_recon_loading.properties` – defines all `$MNAAS*` variables, Hive/Impala connection strings, Java classpath, file extensions, backup directories, log file name, process‑status file name, etc. |
| **External Services** | - Hadoop HDFS (`hadoop fs` commands) <br> - Hive (`hive -e`) <br> - Impala (`impala-shell`) <br> - Java runtime (loader JAR) <br> - Local mail subsystem (`mail`) for SDP ticket/email |
| **Input Files** | - Raw monthly recon files matching `$IPVPROBE_Monthly_Recon_ext` in `$MNAASMainStagingDirDaily` <br> - Process‑status file (`$MNAAS_ipvprobe_Monthly_Recon_ProcessStatusFileName`) |
| **Generated Files** | - Log file (`$MNAAS_ipvprobe_Monthly_recon_loadingLogName`) <br> - Header‑stripped files in `$MNAASInterFilePath_Monthly_IPVProbe_Recon_Details/` <br> - Backup copies in `$Daily_IPVProbe_BackupDir/` |
| **Database Effects** | - Populates temporary Hive table `$ipvprobe_monthly_recon_inter_tblname` <br> - Inserts/overwrites partitions in final Hive table `$ipvprobe_monthly_recon_tblname` <br> - Impala metadata refresh for the final table |
| **State Changes** | - Updates flags (`MNAAS_Monthly_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_job_ran_time`) in the process‑status file <br> - Writes PID (`MNAAS_Script_Process_Id`) to the same file |
| **Assumptions** | - The properties file correctly defines all variables and points to existing directories/HDFS paths. <br> - Hadoop, Hive, Impala, and Java are reachable from the host. <br> - The Java loader JAR (`$Load_nonpart_table`) is compatible with the schema of the temp table. <br> - Mail service is configured and `$GTPMailId`/`$ccList` are valid. |

---

## 4. Integration with Other Scripts / Components

| Connected Component | Interaction |
|---------------------|-------------|
| **`MNAAS_ipvprobe_daily_recon_loading.sh`** | Likely runs earlier in the month to process daily recon files; both scripts share the same property file naming convention and may write to the same intermediate directory. |
| **Cron Scheduler** | The script is invoked by a monthly cron entry (e.g., `0 2 1 * * /path/MNAAS_ipvprobe_monthly_recon_loading.sh`). |
| **Process‑Status File** | Shared with other MNAAS jobs (e.g., other monthly loads) to coordinate sequential execution via the flag values (0‑4). |
| **Backup Scripts** (`MNAAS_*_backup_files_process*.sh`) | The backup step (`MNAAS_cp_monthly_recon_files_to_backup`) mirrors the pattern used by other backup orchestration scripts, storing raw files for audit/re‑run. |
| **Alerting / Ticketing System** | The `email_and_SDP_ticket_triggering_step` integrates with the organization’s SDP ticketing platform via email; other MNAAS scripts use the same mechanism. |
| **Java Loader** (`Load_nonpart_table`) | Shared across multiple load scripts; the same JAR may be invoked by other monthly or daily ingestion jobs. |
| **Hive / Impala Metastore** | The final table (`$ipvprobe_monthly_recon_tblname`) is consumed downstream by reporting/analytics pipelines (e.g., BI dashboards). |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent Execution** – PID guard may fail if the process‑status file is corrupted or the previous PID is stale. | Duplicate loads, data duplication, or table lock contention. | Add a sanity check that the PID truly exists (`ps -p $PID`) and implement a lockfile with `flock`. |
| **Partial File Transfer** – If `hadoop fs -copyFromLocal` succeeds for some files but fails for others, the temp table may be incomplete. | Inconsistent reporting, downstream job failures. | Verify the number of files copied matches the source count; abort on mismatch and move files to a “failed” staging area. |
| **Java Loader Failure** – Exit code not captured beyond `$?` after the `java` command; silent failures could go unnoticed. | Data not loaded, but script proceeds to partition drop. | Capture Java stdout/stderr to a dedicated log, and enforce a non‑zero exit on any error pattern. |
| **Hive Partition Drop / Insert Race** – Dropping partitions before the insert may remove data that another concurrent job is loading. | Data loss. | Ensure exclusive lock on the target table (e.g., via a lock table in Hive) or serialize all monthly loads via the process‑status flag. |
| **Mail/SDP Flooding** – Re‑run of a failed job could generate duplicate tickets. | Alert fatigue. | Persist a ticket ID in the status file and check before sending a new email. |
| **Hard‑coded Permissions (`chmod -R 777`)** – Overly permissive file modes on the intermediate directory. | Security exposure. | Restrict to the minimal required user/group (e.g., `chmod -R 750`). |
| **Missing Environment Variables** – If any property is undefined, the script will abort with cryptic errors. | Unexpected job termination. | Add a validation block after sourcing the properties file that checks for required variables and exits with a clear message. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ipvprobe_monthly_recon_loading.properties` is present and readable.  
   - Verify Hadoop, Hive, Impala, Java, and mail utilities are on the `$PATH`.  
   - Confirm the process‑status file exists and is writable.

2. **Manual Execution**  
   ```bash
   export MNAAS_IPVPROBE_MONTHLY_RECON_SCRIPT_DEBUG=1   # optional, forces verbose logging
   /path/to/MNAAS_ipvprobe_monthly_recon_loading.sh
   ```

3. **Typical Cron Entry**  
   ```cron
   0 3 1 * * /path/to/MNAAS_ipvprobe_monthly_recon_loading.sh >> /var/log/cron_monthly_ipvprobe.log 2>&1
   ```

4. **Debug Steps**  
   - **Check PID Guard**: `cat $MNAAS_ipvprobe_Monthly_Recon_ProcessStatusFileName | grep MNAAS_Script_Process_Id`  
   - **Inspect Log**: Tail the log file defined by `$MNAAS_ipvprobe_Monthly_recon_loadingLogName`.  
   - **Validate HDFS Copy**: `hadoop fs -ls $MNAAS_Monthly_Recon_load_ipvprobe_PathName`  
   - **Verify Hive Tables**: `hive -e "show tables like '${ipvprobe_monthly_recon_tblname}';"`  
   - **Force a Specific Stage**: Manually set `MNAAS_Monthly_ProcessStatusFlag` in the status file to `2` to start from the temp‑table‑load step.  

5. **Exit Codes**  
   - `0` – Successful completion (status file set to *Success*).  
   - `1` – Failure; an email/SDP ticket has been generated.

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_ipvprobe_Monthly_recon_loadingLogName` | Path to the script’s log file. |
| `MNAAS_ipvprobe_Monthly_Recon_ProcessStatusFileName` | Central status/flag file shared across MNAAS jobs. |
| `MNAASMainStagingDirDaily` | Root directory where raw daily files land. |
| `IPVPROBE_Monthly_Recon_ext` | Filename pattern (e.g., `*.ipvprobe_monthly`) for the raw files. |
| `MNAASInterFilePath_Monthly_IPVProbe_Recon_Details` | Intermediate directory for header‑stripped files. |
| `MNAAS_Monthly_Recon_load_ipvprobe_PathName` | HDFS target directory for the `copyFromLocal`. |
| `Load_nonpart_table` | Java class (or wrapper script) that loads CSV into Hive. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath for the loader. |
| `dbname` | Hive database name. |
| `ipvprobe_monthly_recon_inter_tblname` | Temporary Hive table name. |
| `ipvprobe_monthly_recon_tblname` | Final partitioned Hive table name. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive connection details (used by the Java loader). |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `Daily_IPVProbe_BackupDir` | Filesystem backup location for raw files. |
| `MNAAS_ipvprobe_Monthly_Recon_ScriptName` | Human‑readable script identifier used in logs/emails. |
| `GTPMailId`, `ccList` | Email recipients for failure notifications. |
| `Dname_MNAAS_Load_Monthly_ipvprobe_recon_tbl_temp` | Name used for the Java process PID check. |

*If any of the above variables are missing or empty, the script will fail early; a sanity‑check block is recommended.*

---

## 8. Suggested TODO / Improvements

1. **Add a Pre‑Run Validation Block** – After sourcing the properties file, iterate over a required‑variables list and abort with a clear error if any are undefined. This prevents obscure “command not found” failures.

2. **Replace `chmod -R 777` with Secure Permissions** – Use a dedicated group (e.g., `mnaas`) and set directory permissions to `750`. Also consider using ACLs if multiple service accounts need read/write access.

*(Both changes improve reliability and security without altering functional behavior.)*