**Move_Invoice_Register.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`Move_Invoice_Register.sh` is the end‑to‑end ETL driver that ingests daily invoice register files (located in `/Input_Data_Staging/Move_Invoice_Register/SEBS_CHN_DTL_INVCE_REG`), backs them up, strips the first 10 header lines, stages the cleaned files to HDFS, loads the data into a temporary Hive/Impala table (`SEBS_MOB_DTL_INVCE_REG_temp`), and finally merges the records into the production table (`SEBS_MOB_DTL_INVCE_REG`). The script maintains a process‑status file to allow safe restart after failures and sends email/SDP alerts on unrecoverable errors.

---

### 2. Key Functions / Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_cp_files_to_backup** | Copies the incoming file to the backup directory and updates the status flag to `1`. |
| **MNAAS_rm_header_from_files** | Removes the first 10 lines (header) from the source file; sets status flag to `2`. |
| **Move_invoice_register_cp_files_to_hdfs** | Clears the HDFS staging path, copies the cleaned file to HDFS (`MoveInvoiceRegister_dump_PathName`); sets flag to `3`. |
| **Move_invoice_register_load_files_into_temp_table** | Executes the Java loader (`Load_nonpart_table`) to populate the temporary Hive table; refreshes Impala metadata; sets flag to `4`. |
| **Move_invoice_register_insert_into_main_table** | Executes the Java merge class (`invoice_noninvoice_table_loaading`) to upsert data from the temp table into the production table; refreshes Impala metadata; sets flag to `5`. |
| **terminateCron_successful_completion** | Resets the status file to “idle” (`flag=0`), logs success, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | Updates status to `Failure`, sends a notification email to the development team, creates an SDP ticket (via `mailx`), and records the action in the status file. |
| **Main driver block** | Checks for concurrent runs, reads the current flag from the status file, and invokes the functions in the correct order to resume from the last successful step. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Input files** | One (or more) invoice register files placed in `$Move_Invoice_Register` (`/Input_Data_Staging/Move_Invoice_Register/SEBS_CHN_DTL_INVCE_REG`). |
| **Backup** | Files are copied to `$Move_Invoice_Register_BackupDir` (`/backup1/MNAAS/Filesbackup/InvoiceRegister`). |
| **HDFS staging** | `$MoveInvoiceRegister_dump_PathName` (`$MNAASPathName/MoveInvoiceRegister_dump`). |
| **Hive/Impala tables** | Temporary: `SEBS_MOB_DTL_INVCE_REG_temp` <br> Production: `SEBS_MOB_DTL_INVCE_REG` |
| **Process status file** | `$Move_Invoice_Register_ProcessStatusFileName` (path defined in `MNAAS_ShellScript.properties`). Holds flags, job name, timestamps, and email‑ticket flags. |
| **Logs** | `$Move_Invoice_Register_LogPath` (`$MNAASLocalLogPath/Move_Invoice_Register_Log`) – one log file per day (`*_YYYY-MM-DD`). |
| **External services** | - Hadoop HDFS (`hadoop fs`) <br> - Hive/Impala (`impala-shell`) <br> - Java runtime (JARs in `$MNAAS_Main_JarPath`, `$Generic_Jar_Names`) <br> - Mail system (`mail`, `mailx`) for alerts and SDP ticket creation. |
| **Assumptions** | - The properties file supplies all required environment variables (paths, DB credentials, email lists). <br> - Only one file is present in the input directory per run. <br> - Required Java classes are compatible with the current Hive/Impala schema. <br> - The user executing the script has HDFS, Hive, and OS permissions for all paths. |

---

### 4. Integration Points (how it connects to other components)  

| Connection | Description |
|------------|-------------|
| **MNAAS_ShellScript.properties** | Central configuration file shared across all “Move” scripts (e.g., `MNAAS_Traffic_tbl_*`, `MNAAS_Weekly_KYC_Feed_tbl_loading.sh`). Provides DB names, Hadoop namenode, classpath, email lists, etc. |
| **Common Java loader JARs** | Uses the same generic loader (`Load_nonpart_table`) and the invoice‑specific class (`invoice_noninvoice_table_loaading`) as other Move scripts, ensuring consistent data‑loading logic. |
| **Process‑status file** | Same format as other Move scripts; enables the orchestration engine (cron or external scheduler) to monitor progress and restart from the correct step. |
| **HDFS staging directory** | Shared root `$MNAASPathName` used by other Move scripts for temporary dumps, allowing downstream processes (e.g., aggregation jobs) to pick up data. |
| **Impala/Hive metadata refresh** | Calls `refresh` statements identical to other scripts, ensuring that downstream analytical jobs see the latest data. |
| **Alerting / SDP ticketing** | Email and SDP ticket format mirrors the pattern used in `MNAAS_Actives_tbl_Load` and other scripts, feeding into the same incident‑management pipeline. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale status flag** (script crashes before resetting) | Subsequent runs may skip steps or re‑process data, causing duplicates or data loss. | Add a watchdog that clears flags older than a configurable threshold; monitor the status file via a health‑check script. |
| **Missing or multiple input files** | Script may pick the wrong file or fail with `ls` returning multiple names. | Enforce a naming convention and pre‑run validation (`if [ $(ls $dir | wc -l) -ne 1 ]; then abort`). |
| **Backup copy failure** | Original file could be lost if later steps delete it. | Verify copy with checksum (`md5sum`) before deleting source; keep the original until the whole pipeline succeeds. |
| **Header removal removes data rows** (if file has fewer than 10 lines) | Data loss. | Add a guard: `if [ $(wc -l < $file) -gt 10 ]; then sed -i '1,10d' …; else log warning and skip`. |
| **HDFS copy or Java job failure** | Incomplete load, downstream jobs see partial data. | Retry logic with exponential back‑off; alert on first failure; ensure idempotent loads (temp table is truncated each run). |
| **Concurrent executions** | Duplicate processing, lock contention. | The script already checks `ps aux`; consider using a lock file (`flock`) for atomicity. |
| **Email/SDP ticket not sent** (mailx mis‑configured) | Failure goes unnoticed. | Verify mail service health; fallback to writing to a local “alerts” file that a monitoring daemon watches. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Manual execution** | `bash /app/hadoop_users/MNAAS/MNAAS_Property_Files/Move_Invoice_Register.sh` (run as the same user that cron uses). |
| **Set debug mode** | The script already has `set -x`; you can also export `BASH_XTRACEFD=5` to redirect trace to a file. |
| **Check pre‑conditions** | ```bash<br>ls -1 $Move_Invoice_Register<br>cat $Move_Invoice_Register_ProcessStatusFileName<br>``` |
| **Force a specific step** (e.g., start from HDFS copy) | Edit the status file: `sed -i 's/Move_Invoice_Register_ProcessStatusFlag=.*/Move_Invoice_Register_ProcessStatusFlag=3/' $Move_Invoice_Register_ProcessStatusFileName` then run the script. |
| **View logs** | `tail -f $Move_Invoice_Register_LogPath$(date +_%F)` |
| **Validate HDFS data** | `hadoop fs -ls $MoveInvoiceRegister_dump_PathName` |
| **Validate Hive tables** | ```sql<br>impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM $SEBS_MOB_DTL_INVCE_REG;"<br>``` |
| **Check for lingering Java processes** | `ps -ef | grep $Dname_MNAAS_Load_invoice_tbl_temp` |
| **Simulate failure** | Introduce a syntax error or `exit 1` in a function, then verify that the status file is updated to `Failure` and an email/SDP ticket is generated. |

---

### 7. External Configuration & Environment Variables  

| Variable (defined in `MNAAS_ShellScript.properties`) | Use |
|----------------------------------------------------|-----|
| `MNAASConfPath` | Directory containing the process‑status file. |
| `MNAASLocalLogPath` | Base path for daily log files. |
| `MNAASPathName` | Root HDFS staging directory. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath for the loader JARs. |
| `Load_nonpart_table` | Java class name for loading into the temp table. |
| `nameNode`, `dbname`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Hadoop/Hive/Impala connection details. |
| `MNAAS_Main_JarPath` | Path to `mnaas_invoice.jar` containing the merge class. |
| `invoice_noninvoice_table_loaading` | Fully‑qualified Java class for upsert logic. |
| `processname_invoice` | Logical name passed to the Java class (used for logging). |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / sender for alerting. |
| `Move_Invoice_Register_ProcessStatusFile` | Full path to the status file manipulated throughout the run. |

If any of these variables are missing or point to wrong locations, the script will fail early (e.g., `sed` cannot edit the status file, `java` cannot find the JAR).

---

### 8. Suggested Improvements (TODO)  

1. **Add checksum verification for backup** – after copying the source file to `$Move_Invoice_Register_BackupDir`, compute and compare MD5/SHA256 hashes to guarantee the backup is identical before deleting the original.  
2. **Replace `ls | wc -l` concurrency checks with a lock file** – use `flock -n /var/run/Move_Invoice_Register.lock` to guarantee a single instance and avoid race conditions when the `ps` count is unreliable (e.g., when the script name appears in the grep command itself).  

--- 

*End of documentation.*