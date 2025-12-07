**MNAAS_FailedEvents_tbl_Load.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_FailedEvents_tbl_Load.sh` is the daily ingestion pipeline for *FailedEvents* CDR files. It validates incoming files, moves them to staging/backup/reject locations, strips/rewrites headers, copies the cleaned data to HDFS, loads the data into a temporary Hive/Impala table via a Java loader, and finally inserts the rows into the production *FailedEvents* raw table. The script maintains a process‑status file that enables safe restart from the last successful step if the job aborts, and it generates alerts (email + SDP ticket) on failure.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Checks that the newest file(s) in the staging directory match the expected naming pattern and are plain‑text. Moves good files to the *intermediate* directory, bad files to *rejected*, and empty files to *empty* folder. Updates process‑status flag to **1**. |
| **MNAAS_cp_files_to_backup** | Copies all intermediate files to a daily backup directory. Updates process‑status flag to **2**. |
| **MNAAS_rm_header_from_files** | For each intermediate file, removes the original header, re‑creates a pipe‑delimited record set, prefixes each line with the original first column, the filename, and writes the result to a “rm_header” folder. Updates process‑status flag to **3**. |
| **MNAAS_mv_files_to_hdfs** | Deletes any existing files in the target HDFS raw‑table path, then copies the cleaned files from the *rm_header* folder to HDFS. Updates process‑status flag to **4**. |
| **MNAAS_load_files_into_temp_table** | Executes the Java class **Load_nonpart_table** to bulk‑load the HDFS files into a temporary Hive/Impala table (`$FailedEvents_daily_inter_tblname`). Refreshes the Impala metadata. Cleans up intermediate local folders. Updates process‑status flag to **5**. |
| **MNAAS_insert_into_raw_table** | Executes the Java class **Insert_Part_Daily_table** to merge the temporary table into the final raw table (`$FailedEvents_daily_tblname`). Refreshes Impala metadata. Updates process‑status flag to **6**. |
| **terminateCron_successful_completion** | Writes *Success* status, timestamps, and resets the flag to **0** in the process‑status file; logs completion and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, calls **email_and_SDP_ticket_triggering_step**, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email to the MOVE dev team and creates an SDP ticket (via `mailx`). Updates the status file to indicate a ticket has been raised. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input files** | CDR files matching `^[a-zA-Z]+_[0-9]+_[a-zA-Z]+_[0-9]+\.cdr$` located in `$MNAASMainStagingDirDaily_v1/$FailedEvents_extn`. |
| **Intermediate files** | Copied to `$MNAASInterFilePath_Daily_FailedEvents` → cleaned to `$MNAASInterFilePath_Daily_FailedEvents_rm_header`. |
| **Backup** | All intermediate files are duplicated to `$Daily_FailedEvents_BackupDir`. |
| **Rejected/Empty** | Files failing format checks go to `$MNAASRejectedFilePath`; empty files go to `$EmptyFileDir`. |
| **HDFS output** | Files placed under `$MNAAS_Daily_Rawtablesload_FailedEvents_PathName` (raw‑table HDFS location). |
| **Hive/Impala tables** | Temporary table `$FailedEvents_daily_inter_tblname`; final raw table `$FailedEvents_daily_tblname`. |
| **Process‑status file** | `$MNAAS_Daily_FailedEvents_Load_Aggr_ProcessStatusFileName` – holds flags, PID, timestamps, job status, email‑sent flag. |
| **External services** | - Hadoop HDFS (`hadoop fs`) <br> - Hive/Impala (`impala-shell`) <br> - Java runtime (loader & inserter JARs) <br> - Mail system (`mail`, `mailx`) <br> - SDP ticketing endpoint (email to `insdp@tatacommunications.com`) |
| **Assumptions** | • The properties file exists and defines all required variables. <br> • Only one file is processed per run (`No_of_files_to_process=1`). <br> • No other instance of the script is running (PID lock). <br> • Required Java JARs are on `$CLASSPATHVAR` / `$MNAAS_Main_JarPath`. |

---

### 4. Interaction with Other Scripts / Components

| Connected component | How it is used |
|---------------------|----------------|
| **MNAAS_FailedEvents_tbl_Load.properties** | Supplies all directory paths, file extensions, Hive/Impala connection details, Java class names, email lists, etc. |
| **Process‑status file** (`$MNAAS_Daily_FailedEvents_Load_Aggr_ProcessStatusFileName`) | Shared with other daily aggregation scripts (e.g., `MNAAS_Daily_Validations_Checks.sh`) to coordinate restart points and avoid duplicate runs. |
| **Daily staging directory** (`$MNAASMainStagingDirDaily_v1`) | Populated by upstream ingestion/orchestration scripts that pull raw CDRs from network elements or SFTP sources. |
| **Backup & Rejection directories** | Consumed by housekeeping or audit scripts that archive processed files or investigate rejected records. |
| **Java loader/inserter JARs** (`Load_nonpart_table`, `Insert_Part_Daily_table`) | Same JARs are invoked by other table‑load scripts (e.g., for *SimInventory*, *Tolling*). |
| **Impala/Hive metadata refresh statements** (`${FailedEvents_daily_inter_tblname_refresh}`, `${FailedEvents_daily_tblname_refresh}`) | Defined in the properties file; other reporting scripts rely on the refreshed tables. |
| **SDP ticketing / email alerts** | Integrated with the MOVE incident‑management pipeline; failures are also visible in the central monitoring dashboard. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect file naming or non‑text CDRs** | Files moved to *rejected*; downstream tables miss data. | Validate naming conventions upstream; add a pre‑ingest checksum. |
| **HDFS write failure (disk full, permission)** | Job aborts, data not loaded. | Monitor HDFS capacity; ensure the script runs with proper HDFS user; add retry logic. |
| **Java loader/inserter crashes** | Partial load, stale temp table. | Capture Java stack traces in log; enforce idempotent loads; add a cleanup step on failure. |
| **Concurrent execution (PID lock failure)** | Duplicate processing, data duplication. | Verify PID file handling; use a lock file with `flock` for atomic acquisition. |
| **Missing/incorrect properties** | Script exits early or writes to wrong paths. | Add a sanity‑check block at start that verifies all required vars are set. |
| **Email/SDP ticket flood on repeated failures** | Alert fatigue. | Throttle ticket creation (e.g., only once per 24 h) and include a “re‑open” flag. |
| **Hard‑coded `No_of_files_to_process=1`** | New files arriving after the run are ignored until next schedule. | Make the count configurable or process all pending files. |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_FailedEvents_tbl_Load.properties` is present and readable.  
   - Verify environment variables referenced in the properties (e.g., `$CLASSPATHVAR`, `$HIVE_HOST`) are exported.  
   - Confirm Hadoop, Hive, Impala, and Java are reachable from the host.

2. **Typical execution** (called by cron)  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_FailedEvents_tbl_Load.sh
   ```

3. **Manual run (debug mode)**  
   - Uncomment the `set -x` line at the top to enable Bash tracing.  
   - Export `MNAAS_DailyFailedEventsLoadAggrLogPath` to a writable location.  
   - Run the script interactively; monitor the log file for each step.  

4. **Checking status**  
   - Process flag file: `cat $MNAAS_Daily_FailedEvents_Load_Aggr_ProcessStatusFileName` shows current step (`MNAAS_Daily_ProcessStatusFlag`).  
   - PID lock: `ps -p $(grep MNAAS_Script_Process_Id $MNAAS_Daily_FailedEvents_Load_Aggr_ProcessStatusFileName | cut -d= -f2)`.

5. **Isolating a step**  
   - Each function can be invoked directly (e.g., `MNAAS_mv_files_to_hdfs`) after sourcing the script:  
     ```bash
     source MNAAS_FailedEvents_tbl_Load.sh
     MNAAS_mv_files_to_hdfs
     ```
   - This is useful for re‑running a failed step after fixing the root cause.

6. **Log locations**  
   - Primary log: `$MNAAS_DailyFailedEventsLoadAggrLogPath` (appended throughout).  
   - Error files: `$MNAAS_FailedEvents_Error_FileName` (format validation), `$MNAAS_FailedEvents_tbl_Load_Aggr_ProcessStatusFileName` (status).

---

### 7. External Configuration & Environment Variables

| Variable (populated in the `.properties` file) | Meaning / Usage |
|-----------------------------------------------|-----------------|
| `MNAAS_DailyFailedEventsLoadAggrLogPath` | Path to the script’s log file. |
| `MNAAS_Daily_FailedEvents_Load_Aggr_ProcessStatusFileName` | Central status/flag file. |
| `MNAASMainStagingDirDaily_v1` | Root staging directory for daily inbound files. |
| `FailedEvents_extn` | Sub‑directory (or file extension) where *.cdr* files reside. |
| `MNAASInterFilePath_Daily_FailedEvents` | Intermediate processing folder. |
| `MNAASRejectedFilePath` | Destination for malformed files. |
| `EmptyFileDir` | Destination for zero‑byte files. |
| `MNAASInterFilePath_Daily_FailedEvents_rm_header` | Folder holding header‑stripped files. |
| `Daily_FailedEvents_BackupDir` | Daily backup location for original files. |
| `MNAAS_Daily_Rawtablesload_FailedEvents_PathName` | HDFS directory for raw table load. |
| `Load_nonpart_table` / `Insert_Part_Daily_table` | Java class names invoked for load/insert. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components. |
| `dbname`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Hive connection details. |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `FailedEvents_daily_inter_tblname`, `FailedEvents_daily_tblname` | Hive/Impala table names. |
| `FailedEvents_daily_inter_tblname_refresh`, `FailedEvents_daily_tblname_refresh` | SQL statements (e.g., `REFRESH <table>`). |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email notification parameters. |
| `MNAASDailyFailedEventsAggregationScriptName` | Human‑readable script identifier used in logs/emails. |

If any of these variables are missing or empty, the script will likely fail early; a sanity‑check block is recommended.

---

### 8. Suggested Improvements (TODO)

1. **Make the file‑count configurable** – replace the hard‑coded `No_of_files_to_process=1` with a property (e.g., `max_files_per_run`) to allow batch processing when backlog builds up.  
2. **Add robust error handling around external commands** – wrap `hadoop fs`, `java`, and `impala-shell` calls in functions that capture stdout/stderr, retry on transient failures, and write structured error codes to the status file.

---