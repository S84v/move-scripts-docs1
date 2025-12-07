**File:** `move-mediation-scripts/bin/mnaas_tbl_load_generic.sh`  

---

## 1. High‑Level Summary
`mnaas_tbl_load_generic.sh` is the generic driver that orchestrates the end‑to‑end daily load of a Mediation‑North‑America‑Analytics‑Store (MNAAS) table.  
It validates incoming flat files, backs them up, strips headers, prefixes each record with the source filename, removes duplicate rows, stages the data to HDFS, loads it into a temporary Hive/Impala table, then moves the data into the final raw and reject tables.  Progress is recorded in a per‑process status file (`daily_load_raw_process_status.lst`) so that the script can resume from any step after a failure or a manual restart.

---

## 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `init()` | Loads global properties, per‑process properties, sets log file, defines working directories (`staging_dir`, `datapath`, etc.). |
| `MNAAS_files_pre_validation()` | *Pre‑validation*: removes old temp files, builds a list of candidate files, validates file names & MIME type, moves good files to `${staging_dir}/${processname}/temp/${recurring}` and bad files to a reject directory. Also filters empty files and header‑only files. |
| `MNAAS_cp_files_to_backup()` | Copies the validated files to a permanent backup location (`/backup/MNAAS/Customer_CDR/Filesbackup/...`). |
| `MNAAS_rm_header_from_files()` | Strips the first line (header) from each file in the temp directory. |
| `MNAAS_append_filename_to_start_of_records()` | Prefixes each record with the source filename and a configurable delimiter (default `;`). |
| `MNAAS_remove_duplicates_in_the_files()` | De‑duplicates rows in‑place using `awk '!a[$0]++'`. |
| `MNAAS_load_files_into_temp_table()` | Copies the cleaned files to HDFS, then invokes a Java class (`Load_nonpart_table`) to bulk‑load them into a temporary Hive/Impala table. Refreshes the table via `impala-shell`. |
| `MNAAS_insert_into_raw_table()` | Calls a Java class (`RawTableLoading`) to insert data from the temp table into the final raw table, refreshes the table, and cleans the temp directory. |
| `MNAAS_insert_into_reject_table()` | Same as above but for the reject table (records that failed validation earlier). |
| `terminateCron_successful_completion()` | Writes success flags to the status file, logs completion, and exits 0. |
| `terminateCron_Unsuccessful_completion()` | Writes failure flags, triggers email/SDP ticket, and exits 1. |
| `email_and_SDP_ticket_triggering_step()` | Sends a pre‑formatted email to the support mailbox and marks the ticket‑creation flag. |

*Note:* Functions `MNAAS_move_files_to_another_temp_dir` and duplicate‑removal variants are present but commented out – they remain for historical reference.

---

## 3. Execution Flow (Main Script)

1. **Parse options** `-p processname -r recurring -s semfileextension -f filelike`.  
2. **Validate required arguments**; default `semfileextension` to `SEM`.  
3. **Call `init`** → sets up environment.  
4. **Check for already‑running instance** using PID stored in the status file.  
5. **Read `MNAAS_Daily_ProcessStatusFlag`** from the status file to decide the starting step.  
6. **Execute steps sequentially** based on the flag (0‑9). The flag values map directly to the functions listed above, allowing the script to resume after a failure.  
7. **On success** → `terminateCron_successful_completion`.  
8. **On any error** → `terminateCron_Unsuccessful_completion` (email + SDP ticket).  

---

## 4. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • Command‑line args: `processname`, `recurring` (e.g., `daily`), `semfileextension`, `filelike` (glob pattern for data files). <br>• Global property file: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop`. <br>• Process‑specific property file: `${staging_dir}/${processname}/config/${processname}.properties` (defines `staging_dir`, `logdir`, `hdfs_location`, `dbname`, `CLASSPATHVAR`, `Load_nonpart_table`, `MNAAS_Main_JarPath`, etc.). <br>• Files matching `${datapath}/${filelike}` (raw inbound files). |
| **Outputs** | • Log file: `${logdir}/mnaas_tbl_Load_${processname}.log_YYYY-MM-DD`. <br>• Updated status file: `${daily_load_processesfilestatus}` (flags, timestamps, PID). <br>• Backup copies under `/backup/MNAAS/Customer_CDR/Filesbackup/${processname}/Daily_${processname}_BackupDir/`. <br>• Processed files in `${staging_dir}/${processname}/temp/${recurring}` (intermediate). <br>• Reject files in `${staging_dir}/${processname}/reject/`. <br>• Hive/Impala tables: temporary table `${temptable[${processname}]}`, raw table `${rawtable[${processname}]}`, reject table `${rejecttable[${processname}]}`. |
| **Side Effects** | • HDFS writes (`hadoop fs -copyFromLocal`). <br>• Hive/Impala DDL refreshes (`impala-shell`). <br>• Potential email/SDP ticket generation on failure. |
| **Assumptions** | • All required environment variables are defined in the sourced property files. <br>• Java JARs referenced in `CLASSPATHVAR` and `MNAAS_Main_JarPath` are present and compatible with the Hadoop cluster. <br>• The user executing the script has write permission on staging, backup, HDFS, and Hive locations. <br>• The status file is the single source of truth for process coordination. |

---

## 5. Integration with Other Scripts & Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **`mnaas_move_files_from_staging_genric.sh`** | Typically runs **before** this script to move raw files from an upstream staging area into `${staging_dir}/${processname}/data`. |
| **`mnaas_seq_check_for_feed.sh`** | May be used upstream to verify that the expected sequence of files is present before this loader starts. |
| **`mnaas_tbl_load.sh`** | A thin wrapper that calls this generic script with concrete arguments for a specific process (e.g., `IPVProbe`). |
| **`mnaas_parquet_data_load.sh`** | Loads the same raw data into Parquet format after the raw table is populated; may be scheduled after this script completes successfully. |
| **Hadoop / HDFS** | Used for staging files (`hadoop fs -copyFromLocal`) and for temporary table storage. |
| **Hive / Impala** | Target data warehouse; tables are refreshed after each load step. |
| **Java Loader Classes** (`Load_nonpart_table`, `RawTableLoading`) | Perform bulk insertions; they are invoked via `java -cp`. |
| **Monitoring / Alerting** | The status file is read by external cron/monitoring jobs; email/SDP ticketing is triggered on failure. |

---

## 6. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID / Zombie Process** – script thinks another instance is running and aborts. | Data may not be processed for a day. | Add a watchdog that clears the PID if the process no longer exists (e.g., check `ps -p $PID`). |
| **Partial file ingestion due to early termination** – files moved to temp but not loaded. | Inconsistent raw table. | Ensure the status flag is only advanced after each step succeeds; consider atomic move (`mv`) and checksum verification. |
| **Duplicate removal performed in‑place** – original file overwritten before backup. | Data loss if duplicate removal logic is buggy. | Perform duplicate removal on a copy, then replace the original after verification. |
| **Hard‑coded paths** (`/backup/...`, `/app/hadoop_users/...`) limit portability. | Breaks on environment changes. | Externalize all paths into the property files; add validation at start‑up. |
| **Unbounded `ls | wc -l`** for file counting may be slow with many files. | Performance degradation. | Use `find -maxdepth 1 -type f | wc -l` or `stat`‑based counting. |
| **No explicit error handling for Java calls** – only `$?` is checked. | Silent failures if Java exits with non‑zero but script continues. | Capture Java stdout/stderr to log and enforce exit on non‑zero. |
| **Email/SDP ticket flood** if the script repeatedly fails. | Alert fatigue. | Add a throttling mechanism (e.g., only send if last ticket > 1 h ago). |

---

## 7. Running / Debugging the Script  

1. **Typical invocation** (example for IPVProbe daily load):  
   ```bash
   ./mnaas_tbl_load_generic.sh -p IPVProbe -r daily -s SEM -f "IPVProbe_*.txt"
   ```
2. **Prerequisites**  
   * Ensure the global property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop` is readable.  
   * Verify that `${staging_dir}/IPVProbe/config/IPVProbe.properties` exists and defines all required variables (`logdir`, `hdfs_location`, `dbname`, `CLASSPATHVAR`, `Load_nonpart_table`, `MNAAS_Main_JarPath`, etc.).  
   * Confirm Hadoop, Hive, Impala clients are in `$PATH`.  
3. **Debug mode** – the script already runs with `set -x` (trace). To capture full trace:  
   ```bash
   ./mnaas_tbl_load_generic.sh ... 2>&1 | tee /tmp/debug.log
   ```
4. **Inspecting progress**  
   * Tail the log file indicated in the script output (`logpath`).  
   * Check the status file `${daily_load_processesfilestatus}` – the `MNAAS_Daily_ProcessStatusFlag` shows the last completed step.  
5. **Force restart from a specific step** – edit the flag in the status file to the desired value (e.g., `3` to start at “append filename”). Then re‑run the script.  
6. **Common failure points**  
   * **Java class not found** – verify `CLASSPATHVAR` and jar locations.  
   * **HDFS permission denied** – ensure the user has write access to `${hdfs_location}`.  
   * **Impala refresh error** – check Hive metastore connectivity.  

---

## 8. External Config / Environment Variables  

| Variable | Source | Purpose |
|----------|--------|---------|
| `staging_dir` | Process‑specific `.properties` | Root of per‑process staging area. |
| `logdir` | Process‑specific `.properties` | Directory for log files. |
| `daily_load_processesfilestatus` | Process‑specific `.properties` | Status/flag file controlling workflow. |
| `datapath` | Derived from `staging_dir` | Location of inbound raw files (`${staging_dir}/${processname}/data`). |
| `hdfs_location` | Process‑specific `.properties` | Base HDFS path for temporary tables. |
| `dbname` | Process‑specific `.properties` | Hive/Impala database name. |
| `CLASSPATHVAR`, `Load_nonpart_table`, `MNAAS_Main_JarPath` | Global property file | Java classpath and main loader JARs. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Global property file | Connection endpoints for Impala/Hive. |
| `temptable[]`, `rawtable[]`, `rejecttable[]` | Process‑specific `.properties` (associative arrays) | Table names used in loading steps. |
| `chesumvalidationlist` | Global property file | Provides delimiter for filename prefixing. |
| `SDP_*` variables | Global property file | Email addresses and ticketing metadata. |

If any of these are missing, the script will abort with a “command not found” or “undefined variable” error.

---

## 9. Suggested TODO / Improvements  

1. **Refactor duplicate removal to a safe copy‑then‑replace pattern** – prevents data loss if the `awk` command fails mid‑stream.  
2. **Externalize all hard‑coded filesystem paths** (backup, reject, empty‑file dirs) into the per‑process property file and add a validation routine at start‑up to fail fast when a path is missing or unwritable.  

---