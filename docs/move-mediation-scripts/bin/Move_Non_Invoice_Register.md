**Move_Non_Invoice_Register.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`Move_Non_Invoice_Register.sh` orchestrates the nightly ingestion of “non‑invoice” register files into the MNAAS data‑warehouse. It backs up the raw file, strips the first 12 header rows, copies the cleaned file to HDFS, loads it into a temporary Hive/Impala table via a Java loader, and finally merges the data into the production `SEBS_CHN_DTL_INVCE_REG` table. The script maintains a process‑status file to allow safe restart/recovery and sends an SDP ticket + email on failure.

---

### 2. Key Functions / Logical Blocks  

| Function / Block | Responsibility |
|------------------|----------------|
| **MNAAS_cp_files_to_backup** | Updates status flag to **1**, copies the incoming file from the staging directory to the backup directory, logs success/failure. |
| **MNAAS_rm_header_from_files** | Sets flag **2**, removes the first 12 lines (header) from the staged file, logs outcome. |
| **Move_Non_Invoice_Register_cp_files_to_hdfs** | Sets flag **3**, clears the HDFS dump folder, copies the cleaned file to HDFS, logs outcome. |
| **Move_Non_Invoice_Register_load_files_into_temp_table** | Sets flag **4**, launches the Java class `Load_nonpart_table` (via `$Load_nonpart_table` variable) to bulk‑load the HDFS file into the temporary Hive table `SEBS_CHN_DTL_INVCE_REG_temp`. Refreshes the Impala metadata, cleans up local files, logs outcome. |
| **Move_Non_Invoice_Register_insert_into_main_table** | Sets flag **5**, runs the Java class `invoice_noninvoice_table_loaading` to merge the temp table into the production table `SEBS_CHN_DTL_INVCE_REG`. Refreshes Impala metadata, logs outcome. |
| **terminateCron_successful_completion** | Resets status flag to **0**, writes success metadata (run time, job status) to the process‑status file, logs final messages and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Marks job status as *Failure* in the status file, sends an email to the dev team and creates an SDP ticket (via `mailx`) if not already done, updates the status file. |
| **Main driver block** | Checks for concurrent script instances, reads the current flag from the status file, and executes the appropriate subset of steps to resume from the last successful stage. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Input files** | - Raw non‑invoice register file(s) located in `$Move_Non_Invoice_Register` (e.g. `/Input_Data_Staging/Move_Invoice_Register/SEBS_MOB_DTL_INVCE_REG`). <br> - Process‑status file `$Move_Non_Invoice_Register_ProcessStatusFile` (holds flags, timestamps, email‑sent flag). |
| **External services** | - **HDFS** (copy to `$MoveNonInvoiceRegister_dump_PathName`). <br> - **Hive / Impala** (temp & main tables, metadata refresh). <br> - **Java runtime** (loader JARs). <br> - **Mail / SDP ticketing** (mailx, mail). |
| **Outputs** | - Backup copy of the raw file in `$Move_Non_Invoice_Register_BackupDir`. <br> - Cleaned file on HDFS. <br> - Populated temp Hive table `SEBS_CHN_DTL_INVCE_REG_temp`. <br> - Updated production table `SEBS_CHN_DTL_INVCE_REG`. <br> - Log files under `$Move_Non_Invoice_Register_LogPath` (daily). <br> - Updated process‑status file (flags, run time, success/failure). |
| **Side effects** | - Deletion of the local staged file after successful load. <br> - Potential creation of an SDP ticket & email on failure. |
| **Assumptions** | - The staging directory contains exactly one file per run. <br> - Required Hadoop, Hive, Impala clients and Java are installed and reachable. <br> - The process‑status file exists and is writable. <br> - Environment variables defined in `MNAAS_ShellScript.properties` are correct (paths, DB credentials, hostnames, classpath, jar names). <br> - Sufficient disk space in backup, HDFS dump, and local temp directories. |

---

### 4. Interaction with Other Scripts / Components  

| Connected component | Relationship |
|---------------------|--------------|
| **Move_Invoice_Register.sh** | Shares the same staging directory (`/Input_Data_Staging/Move_Invoice_Register`) and similar status‑file handling; runs earlier in the pipeline to place the raw file. |
| **MNAAS_*_tbl_Load_new.sh**, **MNAAS_*_loading.sh** | Follow the same pattern (backup → header removal → HDFS → temp table → main table). The status‑file naming convention (`Move_Non_Invoice_Register_ProcessStatusFile`) mirrors those scripts, enabling a unified monitoring dashboard. |
| **Java loader JARs** (`mnaas_invoice.jar`, generic loader jars) | Invoked via `java -cp $CLASSPATHVAR:$Generic_Jar_Names` and `$MNAAS_Main_JarPath`. They contain the actual bulk‑load logic. |
| **Impala‑shell** | Used to refresh table metadata after each load (`refresh db.table`). |
| **SDP ticketing system** | Email sent to `insdp@tatacommunications.com` with ticket details; the script sets `MNAAS_email_sdp_created` flag to avoid duplicate tickets. |
| **Logging infrastructure** | All `logger -s` calls write to daily log files under `$Move_Non_Invoice_Register_LogPath`. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Stale or corrupted process‑status file** → script may skip steps or loop. | Add a sanity‑check at start (e.g., verify flag is 0‑5 and timestamps are recent). Rotate status files daily. |
| **Concurrent executions** (multiple cron instances). | The script already checks `ps aux|grep -w $MoveInvoiceRegisterScriptName`. Strengthen with a lock file (`flock`) to guarantee exclusivity. |
| **Missing or multiple input files** → `ls` may return unexpected results. | Validate that exactly one file exists; abort with clear error if count ≠ 1. |
| **HDFS space exhaustion** → copy fails, job aborts. | Monitor HDFS usage; add pre‑copy `hdfs dfs -du` check; alert on low space. |
| **Java loader failures (e.g., schema mismatch)** → data not loaded, temp table left dirty. | Capture Java stack trace in log, retain the HDFS file for re‑run, and avoid `rm $Move_Non_Invoice_Register/*` until success confirmed. |
| **Impala refresh failure** → downstream queries see stale data. | Verify exit code of `impala-shell`; retry a limited number of times before aborting. |
| **Backup directory overflow** → old backups deleted, data loss. | Implement a retention policy (e.g., keep last 30 days) and prune old files. |
| **Email/SDP ticket spam** if script repeatedly fails. | Ensure `MNAAS_email_sdp_created` flag is correctly reset on each new run; add rate‑limiting on ticket creation. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Normal execution** (usually via cron) | `bash /app/hadoop_users/Lavanya/MOVE_Invoice_Register/bin/Move_Non_Invoice_Register.sh` |
| **Force a fresh run** (reset flag) | `sed -i 's/Move_Non_Invoice_Register_ProcessStatusFlag=.*/Move_Non_Invoice_Register_ProcessStatusFlag=0/' $Move_Non_Invoice_Register_ProcessStatusFile` then invoke script. |
| **Run with verbose tracing** (already enabled via `set -x`) | Add `bash -x script.sh` or inspect the generated log file `$Move_Non_Invoice_Register_LogPath$(date +_%F)`. |
| **Check current status flag** | `grep Move_Non_Invoice_Register_ProcessStatusFlag $Move_Non_Invoice_Register_ProcessStatusFile` |
| **Inspect last run log** | `less $Move_Non_Invoice_Register_LogPath$(date +_%F)` |
| **Validate HDFS dump** | `hdfs dfs -ls $MoveNonInvoiceRegister_dump_PathName` |
| **Test Java loader manually** (if needed) | `java -Dname=... -cp $CLASSPATHVAR:$Generic_Jar_Names $Load_nonpart_table $nameNode $MoveNonInvoiceRegister_dump_PathName $dbname $SEBS_CHN_DTL_INVCE_REG_temp_tblname $HIVE_HOST $HIVE_JDBC_PORT /tmp/log.txt` |
| **Debug email/SDP** | Verify `mailx` configuration, check `/var/log/mail.log` for delivery status. |

---

### 7. External Configuration / Environment Variables  

| Variable (defined in `MNAAS_ShellScript.properties` or environment) | Use |
|-------------------------------------------------------------------|-----|
| `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASPathName` | Base directories for status file, logs, and dump path. |
| `Move_Non_Invoice_Register` | Staging directory containing the raw file. |
| `Move_Non_Invoice_Register_ProcessStatusFile` | Process‑status file path. |
| `Move_Non_Invoice_Register_LogPath` | Directory for daily log files. |
| `Move_Non_Invoice_Register_BackupDir` | Backup location for raw files. |
| `MoveNonInvoiceRegister_dump_PathName` | HDFS target directory. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `Load_nonpart_table`, `MNAAS_Main_JarPath` | Java classpath and loader JAR locations. |
| `dbname`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Hive/Impala connection details. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / ticketing metadata. |
| `nameNode` | Hadoop NameNode address used by Java loader. |

If any of these variables are missing or point to non‑existent paths, the script will fail early (e.g., `sed` cannot edit the status file).

---

### 8. Suggested Improvements (TODO)

1. **Lock‑file implementation** – Replace the `ps aux | grep` concurrency check with a robust `flock` lock to avoid race conditions when the script is invoked manually while a cron job is running.  
2. **Input validation** – Add a pre‑run sanity check that ensures exactly one file exists in the staging directory and that the file size is > 0; abort with a clear error message otherwise.  

--- 

*End of documentation.*