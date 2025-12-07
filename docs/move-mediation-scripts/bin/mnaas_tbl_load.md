**File:** `move-mediation-scripts/bin/mnaas_tbl_load.sh`  

---

## 1. High‑Level Summary
`mnaas_tbl_load.sh` is the orchestrator for the daily “IPVProbe” (or generic) load pipeline in the MNAAS mediation suite. It validates incoming flat‑files, backs them up, strips headers, prefixes each record with the source filename, removes duplicate rows, stages the cleaned data to HDFS, loads it into a temporary Hive/Impala table, then merges it into the final raw and reject tables. The script maintains a per‑process status file (`daily_load_raw_process_status.lst`) that records the current step, PID, and overall job outcome, enabling safe restart/recovery if the job aborts mid‑run.

---

## 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `init()` | Loads global properties, sets up log file, resolves process‑specific directories (`staging_dir`, `logdir`, `datapath`), and reads the per‑process status file. |
| `MNAAS_files_pre_validation()` | *Pre‑validation*: enumerates incoming files, checks file‑type (text vs. md5), moves correctly‑formatted files to `${staging_dir}/${processname}/temp/${recurring}` and rejects malformed/empty/header‑only files. Updates status flag = 1. |
| `MNAAS_cp_files_to_backup()` | Copies the validated files to a permanent backup location (`/backup1/MNAAS/Customer_CDR/Filesbackup/...`). Updates status flag = 2. |
| `MNAAS_rm_header_from_files()` | Strips the first line (header) from each staged file. Updates status flag = 3. |
| `MNAAS_append_filename_to_start_of_records()` | Prefixes every record with `<filename>;` to retain source traceability. Updates status flag = 4. |
| `MNAAS_remove_duplicates_in_the_files()` | De‑duplicates rows **in‑place** using `awk '!a[$0]++'`. Updates status flag = 6. |
| `MNAAS_load_files_into_temp_table()` | Copies the cleaned files to HDFS, then runs a Java loader (`Load_nonpart_table`) to populate a temporary Hive table (`${temptable[${processname}]}`). Refreshes the table in Impala. Updates status flag = 7. |
| `MNAAS_insert_into_raw_table()` | Executes a Java job (`Insert_Part_Daily_table`) to merge the temp table into the final raw table (`${rawtable[${processname}]}`). Cleans the temp staging area and refreshes Impala. Updates status flag = 8. |
| `MNAAS_insert_into_reject_table()` | Loads any rejected rows (from the reject directory) into a dedicated reject Hive table (`${rejecttable[${processname}]}`). Updates status flag = 9. |
| `terminateCron_successful_completion()` | Resets status flags to *idle* (`0`), records success timestamp, writes final log entry and exits `0`. |
| `terminateCron_Unsuccessful_completion()` | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, exits `1`. |
| `email_and_SDP_ticket_triggering_step()` | Sends a templated email to the MOVE support mailbox and creates an SDP ticket (once per failure). Updates status file to indicate ticket sent. |

*Note:* Several placeholder functions (`MNAAS_move_files_to_another_temp_dir`, `MNAAS_save_partitions`) are commented out but referenced in the main flow for future extensions.

---

## 3. Execution Flow (Main Script)

1. **Parse CLI options** – `-p <processname> -r <recurring> -s <semfileextension> -f <filelike>`.
2. **Initialize** via `init`.
3. **Check for concurrent run** using PID stored in the status file.
4. **Read current flag** (`MNAAS_Daily_ProcessStatusFlag`) and **branch** to the appropriate step(s).  
   - Flag 0/1 → full pipeline (validation → backup → header removal → filename prefix → dedup → load → raw insert → reject insert).  
   - Flags 2‑9 → resume from the step indicated by the flag (allows restart after a failure).  
5. **On success** → `terminateCron_successful_completion`.  
6. **On any error** → `terminateCron_Unsuccessful_completion` → email/SDP ticket.

---

## 4. Configuration & External Dependencies  

| Source | Usage |
|--------|-------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop` | Global environment variables: `staging_dir`, `logdir`, `CLASSPATHVAR`, `Generic_Jar_Names`, `Load_nonpart_table`, `Insert_Part_Daily_table`, `insert_part_daily_reject_table`, `nameNode`, `hdfs_location`, `dbname`, `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `MNAAS_Main_JarPath`, `MNAAS_Daily_ProcessStatusFlag` etc. |
| `${processname}.properties` (under `${staging_dir}/${processname}/config/`) | Process‑specific values: `No_of_files_to_process`, `filevalidationstring`, `md5filevalidationstring`, mappings for `${temptable[${processname}]}`, `${rawtable[${processname}]}`, `${rejecttable[${processname}]}`. |
| `daily_load_raw_process_status.lst` (per‑process) | Persistent status file that stores flags, PID, timestamps, job status, email‑sent flag. Updated throughout the run. |
| Hadoop / HDFS | `hadoop fs -rm -r`, `hadoop fs -copyFromLocal` – required for staging files. |
| Java loader JARs | `Load_nonpart_table`, `Insert_Part_Daily_table`, `insert_part_daily_reject_table`. Must be on the classpath. |
| Impala / Hive CLI | `impala-shell -i … -q "refresh …"` – required to make newly loaded data visible. |
| `/backup1/MNAAS/Customer_CDR/Filesbackup/…` | Permanent backup location for raw files. |
| `mailx` | Sends failure notification email / SDP ticket. Relies on environment variables `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM`. |
| `logger` (syslog) | Writes operational messages to system log and the script‑specific log file. |

*Assumptions*  
- All directories (`staging_dir`, `logdir`, backup, HDFS paths) exist and are writable by the script user.  
- Required Java JARs are compatible with the Hadoop/Hive versions in the environment.  
- The status file is the single source of truth for restart logic; no external orchestration (e.g., Airflow) modifies it.  

---

## 5. Inputs, Outputs, and Side‑Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - Files matching `${filelike}` pattern in `${staging_dir}/${processname}/data/` (e.g., `*.txt`). <br> - Corresponding `.${semfileextension}` semaphore files. |
| **Outputs** | - Cleaned files in `${staging_dir}/${processname}/temp/${recurring}` (used for HDFS load). <br> - Backup copies in `/backup1/.../Daily_${processname}_BackupDir/`. <br> - Records inserted into Hive tables: temporary (`${temptable[...]}`), raw (`${rawtable[...]}`), reject (`${rejecttable[...]}`). |
| **Side‑effects** | - Updates `daily_load_raw_process_status.lst` (flags, PID, timestamps). <br> - Moves malformed/empty files to `${staging_dir}/${processname}/reject/`. <br> - Writes extensive log entries to `${logdir}/mnaas_tbl_Load_${processname}.log_YYYY-MM-DD`. <br> - Sends email/SDP ticket on failure. |
| **External Calls** | Hadoop CLI, Java loader, Impala‑shell, `mailx`, `logger`. |

---

## 6. Integration Points with Other Scripts  

| Connected Script | Interaction |
|------------------|-------------|
| `mnaas_move_files_from_staging_genric.sh` | Populates `${staging_dir}/${processname}/data/` with raw files that `mnaas_tbl_load.sh` later consumes. |
| `mnaas_parquet_data_load.sh` / `mnaas_parquet_data_load_test.sh` | May be downstream consumers that read the raw Hive tables populated by this script and convert them to Parquet for analytics. |
| `mnaas_seq_check_for_feed.sh` | Could be a pre‑step that validates sequence numbers before files reach the staging area. |
| `mnaas_checksum_validation.sh` | Performs checksum verification on the same files; should run **before** this script to guarantee integrity. |
| `customreportgenerator.sh` | Generates reports based on the raw/reject tables that this script loads. |
| `kycalert.sh` | Monitors the status file (`daily_load_raw_process_status.lst`) for failure flags and raises alerts. |
| `dim_date_time_3months.sh` | May be used by downstream reporting jobs that query the tables loaded here. |

The status flag mechanism enables these scripts (or an external scheduler) to determine whether the load succeeded and whether downstream processes can safely start.

---

## 7. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID / orphan lock** – If the script crashes without clearing `MNAAS_Script_Process_Id`, subsequent runs will be blocked. | Load stalls, data backlog. | Implement a watchdog that checks the PID’s age; if > X hours, clear the lock and log a warning. |
| **Partial duplicate removal** – The current `awk '!a[$0]++'` runs **in‑place** but does not write the deduped output back, so duplicates may persist. | Data quality issues. | Redirect output to a temporary file and replace the original (`awk … > tmp && mv tmp $f`). |
| **Hard‑coded paths** (e.g., `/backup1/...`) may not exist on new nodes. | Job failure on new environments. | Externalize backup root path in a property file; add a pre‑flight directory existence check. |
| **Unbounded `ls | wc -l`** for file counting can be slow with many files. | Performance degradation. | Use `find -maxdepth 1 -type f -name "$filelike" | wc -l` or `shopt -s nullglob` with array length. |
| **No explicit error handling for Hadoop/Java commands** – only `$?` is checked after the whole block, which may mask which sub‑command failed. | Difficult debugging. | Capture exit codes after each critical command and log them separately. |
| **Email flood** – Repeated failures could generate many tickets if the status flag is not reset. | Alert fatigue. | Add a cooldown timer (e.g., only send email if last ticket > 24 h ago). |
| **File name parsing assumes a single dot** (`cut -d'.' -f1`). Files with multiple dots will produce wrong semaphore names. | Mis‑matched `.SEM` files, rejected data. | Use parameter expansion `${filename%%.*}` to strip extension safely. |

---

## 8. Running / Debugging the Script  

1. **Typical invocation** (run from the same host where staging directories reside):  
   ```bash
   ./mnaas_tbl_load.sh -p IPVProbe -r daily -s SEM -f "*.txt"
   ```
   - `-p` = process name (must match a directory under `${staging_dir}`).  
   - `-r` = recurring identifier (e.g., `daily`, `hourly`).  
   - `-s` = semaphore file extension (defaults to `SEM`).  
   - `-f` = glob pattern for the data files.

2. **Dry‑run / Verbose**  
   - The script already runs with `set -x` (command tracing).  
   - To capture full trace, redirect stdout/stderr:  
     ```bash
     ./mnaas_tbl_load.sh … > /tmp/mnaas_debug.log 2>&1
     ```

3. **Inspect status**  
   ```bash
   cat ${staging_dir}/${processname}/config/daily_load_raw_process_status.lst
   ```
   Look for `MNAAS_Daily_ProcessStatusFlag=` to know where a previous run stopped.

4. **Force restart from a specific step**  
   - Edit the status file and set `MNAAS_Daily_ProcessStatusFlag` to the desired step number (e.g., `4` to start at “append filename”).  
   - Ensure the PID entry is cleared (`MNAAS_Script_Process_Id=`) before re‑running.

5. **Check Hadoop / Impala**  
   ```bash
   hdfs dfs -ls ${hdfs_location}/${recurring^}/${processname}/
   impala-shell -i $IMPALAD_HOST -q "show tables like '${rawtable[${processname}]}';"
   ```

6. **Log location**  
   - `${logdir}/mnaas_tbl_Load_${processname}.log_YYYY-MM-DD` – tail this file for real‑time progress.  

7. **Common failure points**  
   - Missing semaphore file → file moved to reject.  
   - Java loader exit ≠ 0 → check `${logpath}` for Java stack trace.  
   - HDFS permission error → verify `chmod -R 777` succeeded and the Hadoop user has write access.  

---

## 9. External Config / Environment Variables Referenced  

| Variable | Origin | Purpose |
|----------|--------|---------|
| `staging_dir` | `mnaaspropertiese.prop` | Root of per‑process staging area. |
| `logdir` | `mnaaspropertiese.prop` | Directory for script logs. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | `mnaaspropertiese.prop` | Java classpath for loader JARs. |
| `Load_nonpart_table`, `Insert_Part_Daily_table`, `insert_part_daily_reject_table` | `mnaaspropertiese.prop` | Main class names invoked via `java -cp`. |
| `nameNode`, `hdfs_location` | `mnaaspropertiese.prop` | HDFS connection details. |
| `dbname` | `mnaaspropertiese.prop` | Hive/Impala database name. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | `mnaaspropertiese.prop` | Impala/Hive endpoints. |
| `No_of_files_to_process`, `filevalidationstring`, `md5filevalidationstring` | `${processname}.properties` | Limits and regexes for file selection/validation. |
| `temptable[]`, `rawtable[]`, `rejecttable[]` | `${processname}.properties` | Mapping from process name to Hive table names. |
| `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM` | Environment / property file | Email routing for failure notifications. |

If any of these are missing or point to non‑existent paths, the script will abort early with a logged error.

---

## 10. Suggested TODO / Improvements  

1. **Correct duplicate‑removal logic** – redirect `awk` output to a temporary file and replace the original to guarantee duplicates are removed.  
2. **Externalize all hard‑coded filesystem paths** (backup root, reject dir, empty‑file dir) into the per‑process property file and add a pre‑flight validation step that aborts with a clear message if any required directory is missing or unwritable.  

--- 

*End of documentation.*