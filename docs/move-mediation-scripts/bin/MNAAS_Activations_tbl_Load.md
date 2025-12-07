**MNAAS_Activations_tbl_Load.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_Activations_tbl_Load.sh` is the daily batch driver that ingests raw activation CSV files from the MNAAS staging area, validates their format, removes headers and duplicates, backs them up, moves the cleaned data to HDFS, loads the data into a temporary Hive/Impala table, then populates the raw and aggregation activation tables. The script is state‑ful – it records its progress in a process‑status file so that a failed run can be resumed from the last successful step without re‑processing already‑handled files.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **MNAAS_files_pre_validation** | Scans the staging directory, validates file naming pattern and that files are readable text. Moves good files to the *intermediate* directory, bad files to a *rejected* directory, logs actions, and aborts if no files are found. |
| **MNAAS_cp_files_to_backup** | Copies all intermediate files to a daily backup directory for audit / recovery. |
| **MNAAS_rm_header_from_files** | Strips the first line (header) from every intermediate CSV. |
| **MNAAS_remove_dups_file** | Detects duplicate rows between the current file set and the previously processed file set. Emits only the *difference* (new rows) to a global diff file, updates the “previous‑processed” snapshot, and removes duplicate files. |
| **MNAAS_mv_files_to_hdfs** | Removes any existing raw files in the target HDFS path, then copies the cleaned intermediate files to HDFS and sets permissive ACLs. |
| **MNAAS_load_files_into_temp_table** | Executes a Java loader (`Load_nonpart_table`) that imports the HDFS CSVs into a temporary Hive/Impala table, refreshes the Impala metadata, and cleans the intermediate local directory. |
| **MNAAS_insert_into_raw_table** | Executes a Java inserter (`Insert_Part_Daily_table`) that moves data from the temporary table into the *raw* activation table, then refreshes Impala. |
| **MNAAS_insert_into_aggr_table** *(currently disabled)* | Intended to aggregate raw data into a daily summary table (code commented out). |
| **terminateCron_successful_completion** | Writes a “Success” status to the process‑status file, logs completion, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Writes a “Failure” status, triggers email + SDP ticket, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email and creates an SDP ticket (via `mailx`) if not already done. |
| **Main driver block** | Reads the current flag from the status file, decides which functions to invoke (allowing resume from any step), ensures only one instance runs (PID lock), and starts the pipeline. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Script arguments** | 1️⃣ `Activations_extn` – file extension/pattern (e.g., `*.csv`).<br>2️⃣ `MNASS_Prev_Processed_filepath_Activations` – directory that holds the previously‑processed activation file snapshot. |
| **Configuration source** | `MNAAS_Activations_tbl_Load.properties` (sourced at start). Contains all directory paths, HDFS locations, DB connection strings, classpath, jar names, email lists, etc. |
| **Process‑status file** | `$MNAAS_Daily_Activations_Load_Aggr_ProcessStatusFileName` – a key‑value file that stores flags, PID, timestamps, job status, email‑sent flag, etc. Updated by every function. |
| **File system side‑effects** | • Moves/renames files across staging, intermediate, backup, rejected, empty‑file, and diff directories.<br>• Writes a global diff file (`$Activations_diff_filepath_global`).<br>• Copies files to HDFS (`$MNAAS_Daily_Rawtablesload_Activations_PathName`). |
| **Database side‑effects** | • Loads data into temporary Hive/Impala table (`$activations_daily_inter_tblname`).<br>• Inserts into raw activation table (`$activations_daily_tblname`).<br>• (Potentially) aggregates into `$activations_aggr_daily_tblname` (code commented). |
| **External services** | • Hadoop HDFS (`hadoop fs`, `hdfs dfs`).<br>• Impala (`impala-shell`).<br>• Hive (via JDBC parameters).<br>• Mail system (`mail`, `mailx`).<br>• SDP ticketing system (email to `insdp@tatacommunications.com`). |
| **Assumptions** | • All directories defined in the properties file exist and are writable.<br>• Java runtime and required JARs are present on the node.<br>• Impala/Hive services are reachable and credentials are valid.<br>• Only one instance runs at a time (PID lock works).<br>• Input CSVs follow the naming regex `^[a-zA-Z]+_[0-9]+_[a-zA-Z]+_[0-9]+_[0-9]+.csv$`. |

---

### 4. Interaction with Other Scripts / Components  

| Connected Component | How it connects |
|---------------------|-----------------|
| **MNAAS_Activations_tbl_Load.properties** | Provides all configurable paths, DB hosts, ports, jar names, email lists, etc. |
| **Java loader JARs** (`Load_nonpart_table`, `Insert_Part_Daily_table`) | Invoked from `MNAAS_load_files_into_temp_table` and `MNAAS_insert_into_raw_table`. These JARs are shared across other MNAAS daily load scripts (e.g., for other entities like *Deactivations*). |
| **HDFS raw tables path** (`$MNAAS_Daily_Rawtablesload_Activations_PathName`) | Same HDFS location used by downstream aggregation jobs (e.g., daily reporting, Tableau ingestion). |
| **Impala/Hive tables** (`activations_daily_inter_tblname`, `activations_daily_tblname`) | Populated here; later consumed by reporting pipelines, Tableau data extracts, and possibly by the `tableauMongoSchema.js` script referenced in the history. |
| **Process‑status file** | Shared with other MNAAS daily scripts to coordinate run windows and to expose status to monitoring dashboards. |
| **Email/SDP ticketing** | Uses common mailing lists (`$ccList`, `$MOVE_DEV_TEAM`) and ticketing endpoint used across the MOVE suite. |
| **Other “MNAAS_*_tbl_Load.sh” scripts** | Follow the same pattern (pre‑validation, backup, header removal, duplicate handling, HDFS load, DB load). They may run in parallel for different entities (e.g., *Deactivations*, *Usage*). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect file naming or malformed CSV** | Files are rejected or cause downstream load failures. | Validate naming pattern early (already done) and add schema validation (column count, data types) before HDFS copy. |
| **Duplicate‑detection logic race condition** | If two runs overlap, the “previous‑processed” snapshot may be corrupted, leading to data loss or duplicate loads. | Enforce strict PID lock (already present) and add file‑level locking when updating the snapshot directory. |
| **HDFS permission errors** | `hadoop fs -copyFromLocal` may fail, halting the pipeline. | Ensure the service account has `777` (or appropriate) ACLs on the target path; add a pre‑flight check. |
| **Java loader failures (OOM, classpath issues)** | Stops at temp‑table load, leaving intermediate files on disk. | Capture Java logs, monitor JVM heap, and add retry logic with exponential back‑off. |
| **Impala metadata not refreshed** | Queries see stale data. | Verify `impala-shell` returns success; consider `invalidate metadata` as a fallback. |
| **Email/SDP ticket flood** | Repeated failures could generate many tickets. | Add a throttling flag (already `MNAAS_email_sdp_created`) and ensure it resets only after a successful run. |
| **Process‑status file corruption** | Subsequent runs misinterpret flags. | Store status file on a reliable shared filesystem; optionally checksum after each write. |
| **Hard‑coded `set -x`** | Verbose logs may fill disk. | Rotate log files daily; optionally disable `set -x` in production. |

---

### 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Activations_tbl_Load.properties` is present and all referenced directories exist.  
   - Verify Java, Hadoop, Impala, and mail utilities are in the `$PATH`.  

2. **Typical invocation**  
   ```bash
   ./MNAAS_Activations_tbl_Load.sh .csv /data/mnaas/prev_processed/activations
   ```
   - First argument: file extension/pattern (e.g., `.csv`).  
   - Second argument: absolute path to the directory that holds the previously‑processed activation file snapshot.  

3. **Log location**  
   - All script logs are appended to `$MNAAS_DailyActivationsLoadAggrLogPath` (defined in the properties file).  

4. **Debug steps**  
   - **Check PID lock**: `cat $MNAAS_Daily_Activations_Load_Aggr_ProcessStatusFileName | grep MNAAS_Script_Process_Id`.  
   - **Inspect status flag**: `grep MNAAS_Daily_ProcessStatusFlag $MNAAS_Daily_Activations_Load_Aggr_ProcessStatusFileName`.  
   - **Run with trace** (already enabled via `set -x`) or temporarily add `set -e` to abort on first error.  
   - **Validate intermediate files**: list `$MNAASInterFilePath_Daily_Activations` after each step.  
   - **Java job logs**: the Java commands write to `$MNAAS_DailyActivationsLoadAggrLogPath`; search for `ERROR` or stack traces.  
   - **HDFS verification**: `hadoop fs -ls $MNAAS_Daily_Rawtablesload_Activations_PathName`.  

5. **Manual recovery**  
   - If a step fails, edit the status file to set `MNAAS_Daily_ProcessStatusFlag` to the previous step’s number and re‑run the script.  
   - Ensure no stray PID remains (`ps -ef | grep MNAAS_Activations_tbl_Load.sh`).  

---

### 7. External Configurations & Environment Variables  

| Variable (from properties) | Meaning |
|----------------------------|---------|
| `MNAAS_Daily_Activations_Load_Aggr_ProcessStatusFileName` | Central status/flag file. |
| `MNAAS_DailyActivationsLoadAggrLogPath` | Path to the aggregated log file. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Root staging directory where raw activation files land. |
| `Activations_extn` (overridden by arg) | File extension/pattern to filter files. |
| `MNAASInterFilePath_Daily_Activations` | Intermediate directory for validated files. |
| `MNAASRejectedFilePath` | Directory for malformed/rejected files. |
| `MNAASBackupFilePath` / `Daily_Activations_BackupDir` | Backup locations. |
| `MNAAS_Dups_Handlingpath_Activations` | Temp folder for duplicate‑handling sorting. |
| `Activations_diff_filepath` / `Activations_diff_filepath_global` | Files that hold row‑level differences. |
| `MNAAS_Daily_Rawtablesload_Activations_PathName` | HDFS target path for raw CSVs. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components. |
| `Load_nonpart_table`, `Insert_Part_Daily_table` | Main Java class names invoked. |
| `dbname`, `activations_daily_inter_tblname`, `activations_daily_tblname` | Hive/Impala database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` | Connection endpoints. |
| `ccList`, `GTPMailId`, `MOVE_DEV_TEAM`, `SDP_ticket_from_email` | Email recipients / ticketing metadata. |
| `MNAASDailyActivationsAggregationScriptName` | Human‑readable script identifier used in logs/emails. |

---

### 8. Suggested Improvements (TODO)

1. **Add schema validation** – before moving files to HDFS, parse a few rows to ensure column count and basic data‑type sanity; reject files that fail validation to avoid downstream load errors.  
2. **Externalize the duplicate‑detection algorithm** – replace the current `sort` + `comm` approach with a Spark job or Hive `INSERT OVERWRITE` that can handle larger file volumes more efficiently and reduce reliance on local temporary sorting directories.  

--- 

*End of documentation.*