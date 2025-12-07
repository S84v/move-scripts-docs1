**MNAAS_SimInventory_tbl_Load.sh – High‑Level Documentation**  

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_SimInventory_tbl_Load.sh` is the daily orchestration driver that validates, stages, de‑duplicates, enriches and loads SIM‑Inventory CSV files into the Hadoop ecosystem (HDFS → Hive/Impala) and finally into the production raw and aggregation tables. It maintains a persistent process‑status file to allow safe restart/recovery, backs up the original file, and raises SDP tickets + email alerts on failure.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Picks the newest file from the *sequence‑check* staging area, validates its naming pattern, moves it to the intermediate directory or to the rejected folder, logs the action, and aborts if no file is present. |
| **MNAAS_cp_files_to_backup** | Copies the validated file to a daily backup directory and updates the process‑status flag. |
| **MNAAS_rm_header_from_files** | Strips the first line (CSV header) from the intermediate file. |
| **MNAAS_remove_dups_file** | Detects duplicate records by comparing the current file with the previously processed file; keeps only new rows, updates the “previous file” store, and logs whether the file is a duplicate set. |
| **MNAAS_append_filename_to_start_of_records** | Prefixes each record with the source filename (`filename;`) to preserve provenance. |
| **MNAAS_mv_files_to_hdfs** | Removes any stale files in the target HDFS raw‑table path and copies the processed CSV into HDFS. |
| **MNAAS_load_files_into_temp_table** | Executes a Java loader JAR (`Load_nonpart_table`) that creates a temporary Hive table from the HDFS file, then refreshes the Impala metadata. |
| **MNAAS_insert_into_raw_table** | Executes a Java inserter JAR (`Insert_Part_Daily_table`) that moves data from the temp table into the permanent raw SIM‑Inventory Hive table, followed by an Impala refresh. |
| **MNAAS_insert_into_aggr_table** | (currently commented out) – would load data from the raw table into an aggregation table. |
| **terminateCron_successful_completion** | Writes a “Success” status to the process‑status file, logs completion, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Updates status to “Failure”, sends an email to the MOVE dev team, and creates an SDP ticket via `mailx`. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input files** | • One CSV file placed in `${MNAASMainStagingDirDaily_afr_seq_check}/${SimInventory_extn}` (name must match `YYYY-MM-DD_HH-MM_<partner>_SimInventory_YYYYMM.csv`). <br>• Optional previous‑day file in `${MNASS_Prev_Processed_filepath_SimInventory}` for duplicate detection. |
| **Configuration** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`. This file defines all path variables, Hive/Impala hosts, JAR locations, process‑status file name, email lists, etc. |
| **External services** | • HDFS (`hadoop fs` commands). <br>• Hive/Impala (via Java JARs and `impala-shell`). <br>• Mail system (`mail`, `mailx`). <br>• SDP ticketing system (email to `insdp@tatacommunications.com`). |
| **Outputs** | • Processed CSV in `${MNAASInterFilePath_Daily_SimInventory}` (temporary). <br>• Backup copy in `${Daily_SimInventory_BackupDir}`. <br>• HDFS file under `${MNAAS_Daily_Rawtablesload_SimInventory_PathName}`. <br>• Populated Hive tables (`siminventory_daily_inter_tblname`, `siminventory_daily_tblname`). <br>• Updated process‑status file (`${MNAAS_Daily_SimInventory_Load_Aggr_ProcessStatusFileName}`). |
| **Side‑effects** | • Log entries appended to `${MNAAS_DailySimInventoryLoadAggrLogPath}<date>`. <br>• Potential creation of an SDP ticket & email on failure. |
| **Assumptions** | • Only one file arrives per day. <br>• The naming convention never changes. <br>• Sufficient disk space in staging, backup, and HDFS directories. <br>• Java JARs are compatible with the current Hive/Impala schema. <br>• No concurrent runs (checked via `ps aux | grep -w $MNAASDailySimInventoryAggregationScriptName`). |

---

### 4. Interaction with Other Scripts / Components  

| Connected Script | Role in the overall pipeline |
|------------------|------------------------------|
| **MNAAS_SimInventory_seq_check.sh** | Performs the *sequence* validation that places the file into `${MNAASMainStagingDirDaily_afr_seq_check}` – the entry point for this loader. |
| **MNAAS_SimInventory_backup_files_process.sh** | May be invoked separately to archive older backup files; shares the same backup directory variables. |
| **MNAAS_PreValidation_Checks.sh**, **MNAAS_RawFileCount_Checks.sh**, **MNAAS_SimChange_seq_check.sh**, etc. | Provide generic pre‑validation, file‑count, and change‑detection utilities that use the same process‑status file pattern. |
| **Java JARs** (`Load_nonpart_table`, `Insert_Part_Daily_table`) | Executed by this script to move data from HDFS into Hive/Impala. |
| **Impala/Hive Metastore** | Receives `REFRESH` statements after each load to make data visible to downstream reporting jobs. |
| **SDP ticketing / email** | Integrated via `email_and_SDP_ticket_triggering_step`. |

The script therefore sits in the *daily* branch of the Move mediation pipeline, downstream of file‑arrival checks and upstream of reporting/analytics jobs that consume the raw and aggregation tables.

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect file naming** – script moves file to *rejected* folder and aborts. | Data loss / missed load. | Add a pre‑alert (email) when a file lands in the rejected folder; monitor the rejected directory. |
| **Duplicate‑file handling logic** – relies on a single previous file; race conditions may cause false duplicates. | Missing new records. | Keep a rolling hash or checksum list of processed files; archive previous files with timestamps. |
| **Process‑status flag corruption** – manual edit could break restart logic. | Stuck pipeline. | Protect the status file with OS permissions (read‑only for non‑root) and add a health‑check script that validates flag values. |
| **Concurrent runs** – script checks `ps` but may miss fast‑starting overlapping instances. | Double processing, file contention. | Use a lock file (`flock`) instead of `ps` check. |
| **Java JAR failures** – exit codes captured but no retry. | Partial load, downstream jobs fail. | Implement a retry loop with exponential back‑off; capture JAR stdout/stderr to logs. |
| **HDFS copy failure** – network glitch may leave stale files. | Inconsistent data. | Verify file checksum after `copyFromLocal`; clean up on failure. |
| **Impala refresh failure** – may leave tables invisible. | Reporting errors. | Check return code of `impala-shell`; if non‑zero, trigger alert and abort. |
| **Email/SDP ticket flood** – repeated failures could generate many tickets. | Alert fatigue. | Add a throttling mechanism (e.g., only send one ticket per hour per script). |

---

### 6. Running / Debugging the Script  

1. **Typical execution** – scheduled via cron (e.g., `0 2 * * * /path/MNAAS_SimInventory_tbl_Load.sh`).  
2. **Manual run** –  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./MNAAS_SimInventory_tbl_Load.sh
   ```
   Observe console output (the script runs with `set -x` so each command is echoed).  
3. **Check status** – inspect the process‑status file:  
   ```bash
   cat $MNAAS_Daily_SimInventory_Load_Aggr_ProcessStatusFileName
   ```  
   The flag (`MNAAS_Daily_ProcessStatusFlag`) indicates the last completed step.  
4. **Log inspection** – tail the daily log:  
   ```bash
   tail -f $MNAAS_DailySimInventoryLoadAggrLogPath$(date +_%F)
   ```  
5. **Debugging duplicate logic** – verify the contents of the previous‑file directory and the diff file:  
   ```bash
   ls $MNASS_Prev_Processed_filepath_SimInventory
   cat $SimInventory_diff_filepath
   ```  
6. **Force re‑run from a specific step** – edit the status file to set `MNAAS_Daily_ProcessStatusFlag` to the desired step number (e.g., `3` to start at header removal). Ensure no other instance is running.  

---

### 7. External Configuration / Environment Variables  

| Variable (defined in `MNAAS_ShellScript.properties`) | Usage |
|----------------------------------------------------|-------|
| `MNAAS_Daily_SimInventory_Load_Aggr_ProcessStatusFileName` | Central status file updated by every function. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Source directory where the seq‑check script drops the file. |
| `SimInventory_extn` | Extension/pattern for the incoming CSV (e.g., `*.csv`). |
| `MNAASInterFilePath_Daily_SimInventory` | Working directory for validated files. |
| `MNAASRejectedFilePath` | Destination for malformed files. |
| `EmptyFileDir` | Holds empty files removed during pre‑validation. |
| `Daily_SimInventory_BackupDir` | Daily backup location. |
| `MNAAS_Daily_Rawtablesload_SimInventory_PathName` | HDFS target path. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath and JAR locations for loaders/inserters. |
| `Load_nonpart_table`, `Insert_Part_Daily_table` | Main class names invoked via `java -cp`. |
| `dbname`, `siminventory_daily_inter_tblname`, `siminventory_daily_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `siminventory_daily_inter_tblname_refresh`, `siminventory_daily_tblname_refresh` | Impala `REFRESH` statements (passed as strings). |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients for alerts. |
| `MNAASDailySimInventoryAggregationScriptName` | Script name used for process‑status logging and PID check. |

If any of these variables are missing or point to non‑existent paths, the script will fail early (usually with a `sed` or `cp` error). Verify the property file before first run.

---

### 8. Suggested Improvements (TODO)

1. **Replace `ps | grep` concurrency guard with a robust lock file (`flock`)** – eliminates race conditions and simplifies the “already running” check.  
2. **Add checksum‑based duplicate detection** – generate an MD5/SHA1 of each incoming file and store it alongside the previous file; this avoids costly `comm`/`sort` on large CSVs and guarantees true duplicate detection even when row order changes.  

--- 

*End of documentation.*