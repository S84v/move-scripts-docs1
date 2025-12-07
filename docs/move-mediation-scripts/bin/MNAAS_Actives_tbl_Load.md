**File:** `move-mediation-scripts/bin/MNAAS_Actives_tbl_Load.sh`  

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily “Actives” data‑load pipeline for the MNAAS (Mobile Number Activation & Service) domain. It validates incoming CSV files, backs them up, strips headers, stages data in HDFS, loads the most recent file into a temporary Hive/Impala table, inserts the data into raw tables, removes duplicate records across days, and finally aggregates the data into the production “Actives” table. The script is resumable – a process‑status file stores a flag and the last‑executed step, allowing the job to continue after a failure or manual restart.

---

## 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Checks file naming pattern, ensures files are readable text, moves good files to an intermediate directory, rejects malformed files, and discards empty‑header‑only files. |
| **MNAAS_cp_files_to_backup** | Copies the validated intermediate files to a daily backup directory. |
| **MNAAS_rm_header_from_files** | Removes the first line (header) from each intermediate CSV. |
| **MNAAS_cp_files_to_temp_location** | Copies files to a temporary “dump” area, appends the filename as a prefix to each line, and preserves the previous day’s file for “last‑file‑of‑the‑day” processing. |
| **MNAAS_cp_files_to_hdfs_to_load_last_file_oftheday** | Uploads the dump directory to HDFS (`*_dumpOfLastLoad_PathName`). |
| **MNAAS_load_files_into_temp_table_last_file_oftheday** | Executes a Java JAR (`Load_nonpart_table`) to load the last‑file‑of‑the‑day dump into a temporary Hive table, then refreshes the Impala metadata. |
| **MNAAS_insert_into_raw_table_oftheday** | Executes a Java JAR (`Insert_Part_Daily_table`) to insert the temporary data into the raw “Actives” table for the last file, followed by Impala refresh. |
| **MNAAS_remove_dups_file** | Performs a day‑over‑day duplicate detection: sorts current & previous files, uses `comm` to keep only new rows, prefixes each line with the source filename, and updates the “previous file” store. |
| **MNAAS_mv_files_to_hdfs** | Moves the de‑duplicated files to the final HDFS raw‑load directory. |
| **MNAAS_load_files_into_temp_table** | Loads all de‑duplicated files (now in HDFS) into a temporary Hive table via the same Java loader. |
| **MNAAS_insert_into_raw_table** | Inserts the temporary data into the final raw “Actives” Hive table. |
| **MNAAS_insert_into_aggr_table** *(currently commented out)* | Intended to load data into an aggregation table. |
| **terminateCron_successful_completion** | Writes success status to the process‑status file, logs completion, and exits `0`. |
| **terminateCron_Unsuccessful_completion** | Writes failure status, triggers email/SDP ticket, and exits `1`. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email and creates an SDP ticket (via `mailx`). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | 1. Two positional arguments: <br>  • `$1` – file extension (e.g., `csv`) <br>  • `$2` – path to previous‑processed file directory. <br>2. All variables sourced from `MNAAS_Actives_tbl_Load.properties` (paths, DB names, Hive/Impala hosts, Java classpaths, JAR names, log file locations, email lists, etc.). |
| **Outputs** | 1. Process‑status file (`$MNAAS_Daily_Actives_Load_Aggr_ProcessStatusFileName`) – flag, step name, PID, timestamps, job status. <br>2. Log file (`$MNAAS_DailyActivesLoadAggrLogPath`). <br>3. Files moved to various directories: <br>  • Validated intermediate (`$MNAASInterFilePath_Daily_Actives`) <br>  • Backup (`$Daily_Actives_BackupDir`) <br>  • Empty‑header dir (`$EmptyFileDir`) <br>  • Rejected dir (`$MNAASRejectedFilePath`) <br>  • HDFS raw load paths (`$MNAAS_Daily_Rawtablesload_Actives_PathName`, `*_dumpOfLastLoad_PathName`). |
| **Side Effects** | • HDFS writes/removals (`hadoop fs -copyFromLocal`, `-rm`). <br>• Hive/Impala table loads via Java JARs and `impala-shell` refreshes. <br>• System `logger` entries (syslog). <br>• Email & SDP ticket generation on failure. |
| **Assumptions** | • All directories exist and are writable by the script user. <br>• Hadoop, Hive, Impala services are reachable and the user has required permissions. <br>• Java JARs (`Load_nonpart_table`, `Insert_Part_Daily_table`) are compatible with the supplied classpath. <br>• File naming convention: `<prefix>_<numeric>_<suffix>_<numeric>_<date>.csv`. <br>• No concurrent runs (checked via PID stored in status file). |

---

## 4. Interaction with Other Components  

| Component | Connection Point |
|-----------|------------------|
| **`MNAAS_Actives_tbl_Load.properties`** | Sourced at the top; defines all environment variables used throughout the script. |
| **Hadoop HDFS** | Used for staging (`$MNAASInterFilePath_Daily_Actives*`), raw load directories, and dump of last file. |
| **Hive / Impala** | Temporary tables (`$actives_inter_raw_daily_at_eod_tblname`, `$actives_daily_inter_tblname`, etc.) are created/loaded by the Java JARs; metadata refreshed via `impala-shell`. |
| **Java Loader JARs** (`Load_nonpart_table`, `Insert_Part_Daily_table`) | Called with many arguments (namenode, HDFS path, DB, table names, hosts, ports, log path). |
| **Mail / SDP ticket system** | Failure notifications sent via `mail` and `mailx` to distribution lists and the SDP ticketing address. |
| **Other scripts** | This driver is typically invoked by a cron or scheduler after the “Actives sequence check” script (`MNAAS_Actives_seq_check.sh`). The status file is also read/written by those scripts to coordinate the overall pipeline. |
| **Logging / Monitoring** | Uses `logger -s` to write to syslog; log file is also a primary artifact for operators. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Incorrect file naming or malformed CSV** | Files may be rejected or cause downstream load failures. | Validate naming pattern early (already done) and add a checksum/size check; archive rejected files for manual review. |
| **Concurrent executions** | Two runs could corrupt the status file or duplicate loads. | Keep the PID check (already present) and enforce exclusive lock file (`flock`) before script start. |
| **HDFS permission or space exhaustion** | `hadoop fs -copyFromLocal` may fail, leaving partial data. | Pre‑flight HDFS quota/space check; exit with clear error if insufficient. |
| **Java JAR failure (class‑path, version mismatch)** | Load steps abort, leaving tables partially populated. | Capture JAR exit codes (already done) and add a retry wrapper; maintain a version‑controlled JAR repository. |
| **Duplicate‑removal logic errors** | May drop legitimate rows or keep duplicates. | Add unit tests for the `comm`‑based diff; log counts of rows before/after; optionally keep a “diff audit” file. |
| **Email/SDP ticket flood** | Repeated failures could generate many tickets. | Throttle notifications (e.g., only once per hour) and include a “ticket already created” flag (already in script). |
| **Hard‑coded numeric limits (e.g., `No_of_files_to_process=100`)** | New batches >100 files will be ignored. | Make the limit configurable via the properties file. |
| **Missing or malformed properties file** | Script will exit with undefined variables. | Add a sanity check after sourcing the properties file; abort with a clear error if required vars are empty. |

---

## 6. Running / Debugging the Script  

1. **Preparation**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Actives_tbl_Load.properties` is present and contains all required variables.  
   - Verify that the staging directory `$MNAASMainStagingDirDaily_afr_seq_check` contains the expected CSV files.  
   - Confirm HDFS, Hive, Impala, and Java JARs are reachable.  

2. **Typical Invocation (from cron or manually)**  
   ```bash
   ./MNAAS_Actives_tbl_Load.sh csv /path/to/prev_processed_dir
   ```
   - `$1` = file extension (`csv`).  
   - `$2` = directory that holds the previous day’s processed file (used for duplicate detection).  

3. **Debug Mode**  
   - The script starts with `set -x`, which prints each command as it runs.  
   - To capture full debug output:  
     ```bash
     ./MNAAS_Actives_tbl_Load.sh csv /path/to/prev > /tmp/debug.log 2>&1
     ```  
   - Review `/tmp/debug.log` and the log file defined by `$MNAAS_DailyActivesLoadAggrLogPath`.  

4. **Checking Status**  
   - The process‑status file holds the current flag and PID.  
   - Example: `grep MNAAS_Daily_ProcessStatusFlag $MNAAS_Daily_Actives_Load_Aggr_ProcessStatusFileName` shows where the job will resume on next run.  

5. **Force Restart**  
   - If the script is stuck (e.g., stale PID), delete the PID line from the status file or set the flag to `0` manually, then re‑run.  

6. **Post‑run Validation**  
   - Verify that the target Hive tables contain the expected row counts (`impala-shell -q "SELECT COUNT(*) FROM <table>"`).  
   - Check HDFS directories for the presence of the processed files.  

---

## 7. External Config / Environment Variables  

All variables are defined in `MNAAS_Actives_tbl_Load.properties`. Key ones (non‑exhaustive) include:

| Variable | Purpose |
|----------|---------|
| `MNAAS_DailyActivesLoadAggrLogPath` | Path to the main log file. |
| `MNAAS_Daily_Actives_Load_Aggr_ProcessStatusFileName` | Process‑status control file. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Directory where raw inbound files land. |
| `MNAASInterFilePath_Daily_Actives` | Intermediate “validated” directory. |
| `MNAASRejectedFilePath` | Destination for rejected files. |
| `MNAASInterFilePath_Daily_ActivesDump` | Dump area for “last‑file‑of‑the‑day”. |
| `MNAAS_Daily_Rawtablesload_Actives_PathName` | HDFS path for final raw load. |
| `MNAAS_Daily_Rawtablesload_Actives_dumpOfLastLoad_PathName` | HDFS path for the dump of the last file. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath and JAR locations. |
| `Load_nonpart_table`, `Insert_Part_Daily_table` | Main Java class names invoked. |
| `dbname`, `actives_inter_raw_daily_at_eod_tblname`, `actives_raw_daily_at_eod_tblname`, etc. | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Hive/Impala connection endpoints. |
| `ccList`, `GTPMailId`, `MOVE_DEV_TEAM`, `SDP_ticket_from_email` | Email distribution lists for notifications. |
| `MNAASDailyActivesAggregationScriptName` | Human‑readable script name used in logs/emails. |

If any of these are missing or empty, the script will fail silently; a sanity‑check after sourcing the properties file is advisable.

---

## 8. Suggested Improvements (TODO)

1. **Parameterise the “max files to process” limit** – move `No_of_files_to_process=100` into the properties file so the pipeline can handle larger daily volumes without code changes.  

2. **Add a pre‑flight health check** – a function that validates the existence and write‑access of all required directories, HDFS connectivity, and that required Java JARs are present before any processing begins. This would produce a clearer early‑failure message rather than later “file not found” errors.  

---