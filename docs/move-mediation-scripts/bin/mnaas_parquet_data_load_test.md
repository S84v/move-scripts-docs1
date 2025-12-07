**File:** `move-mediation-scripts/bin/mnaas_parquet_data_load_test.sh`  

---

## 1. High‑Level Summary
This script is a test‑oriented driver for the **MNAAS daily data‑load pipeline**.  
Given a *process name* (e.g., a specific mediation feed), it orchestrates the end‑to‑end flow:

1. **Pre‑validation** of incoming flat files (format, checksum, emptiness).  
2. **Backup** of the validated files.  
3. **Header removal**, **filename‑column injection**, and **duplicate elimination**.  
4. **Conversion to Parquet** using a Java utility and loading the Parquet files into a Hive temporary table.  
5. **Insertion of the data** from the temporary table into the final raw Hive/Impala table (and optionally a reject table).  

The script maintains a **process‑status file** (`daily_load_raw_process_status.lst`) that records a numeric flag and the current step name, enabling restartability and preventing concurrent executions.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| `init()` | Loads global properties, sets up log file, resolves directories (`staging_dir`, `logdir`, `datapath`), and prepares the status‑file path. |
| `MNAAS_files_pre_validation()` | *a)* Marks flag = 1, *b)* Lists candidate files, *c)* Validates file type (text) and naming pattern, *d)* Moves good files to `${staging_dir}/${processname}/temp/${recurring}` and bad/empty files to a reject directory, *e)* Detects “header‑only” files and rejects them. |
| `MNAAS_cp_files_to_backup()` | Copies the validated files to a configured backup location; flag = 2. |
| `MNAAS_append_filename_to_start_of_records()` | Adds a `filename` column as the first field of every record; flag = 4. |
| `MNAAS_remove_duplicates_in_the_files()` | De‑duplicates rows within each file using `awk '!a[$0]++'`; flag = 6. |
| `MNAAS_load_files_into_temp_table()` | For each file: <br>• Runs `ParquetConverterUtility` (Java) to produce a `.parquet` file. <br>• Copies the Parquet files to HDFS (`hdfs_parquet_location/...`). <br>• Repairs the Hive external table (`msck repair`). Flag = 7. |
| `MNAAS_insert_into_raw_table()` | Executes a Java loader (`RawTableLoading`) that inserts the Hive temporary table into the final raw table; refreshes Impala metadata. Flag = 8. |
| `MNAAS_insert_into_reject_table()` | (currently commented out) Loads rejected rows into a dedicated reject table; flag = 9. |
| `terminateCron_successful_completion()` | Resets status flags to *0* (Success), writes timestamps, logs completion, and exits 0. |
| `terminateCron_Unsuccessful_completion()` | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, and exits 1. |
| `email_and_SDP_ticket_triggering_step()` | Sends an email (via `mailx`) to the support team and creates an SDP ticket if not already done. |

> **Note:** The script also references two functions that are **not defined in this file** – `MNAAS_rm_header_from_files` and `MNAAS_move_files_to_another_temp_dir`. They are defined in other pipeline scripts (e.g., `mnaas_parquet_data_load.sh`) and are expected to be available in the runtime environment.

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Command‑line arguments** | `-p <processname>` (mandatory) – logical name of the feed.<br>`-r <recurring>` – sub‑directory identifier (e.g., `daily`).<br>`-s <semfileextension>` – extension for the accompanying SEM file (default `SEM`).<br>`-f <filelike>` – glob/pattern for source files (e.g., `*.txt`). |
| **External configuration files** | • `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop`  <br>• `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed_tbl_Load.properties`  <br>• `${staging_dir}/${processname}/config/${processname}.properties` (process‑specific). |
| **Environment / global vars (populated by the property files)** | `staging_dir`, `logdir`, `backuplocation`, `hdfs_parquet_location`, associative arrays `temptable[]`, `rawtable[]`, `rejecttable[]`, `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `Dname_MNAAS_Insert_Daily_KYC_tbl`, `MNAAS_Daily_KYC_Feed_Load_LogPath`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`, `dbname`, `logpath`, etc. |
| **Primary input data** | Flat files matching `${filelike}` located under `${staging_dir}/${processname}/data`. |
| **Generated artefacts** | • Validated files in `${staging_dir}/${processname}/temp/${recurring}`.<br>• Backup copies under `${backuplocation}/${processname}/Daily_${processname}_BackupDir/`.<br>• Parquet files in HDFS (`$hdfs_parquet_location/...`).<br>• Hive temporary table (`default.${temptable[processname]}`) and raw table (`default.${rawtable[processname]}`). |
| **Side effects** | • Writes to the status file `${daily_load_processesfilestatus}` (flags, timestamps, PID).<br>• Logs extensively to `${logpath}`.<br>• Sends email / creates SDP ticket on failure.<br>• Executes Hadoop/Hive/Impala commands and a Java JAR. |
| **Assumptions** | • All directories exist and are writable by the script user.<br>• Hadoop, Hive, Impala, Java, and `mailx` are installed and reachable.<br>• The Java class `com.tcl.parquet.utility.ParquetConverterUtility` and the loader JARs are present at the paths referenced.<br>• The status file is the single source of truth for process coordination. |

---

## 4. Integration Points & System Context  

| Component | Interaction |
|-----------|-------------|
| **`mnaas_parquet_data_load.sh`** (production counterpart) | Shares the same function names (`MNAAS_*`) and status‑file logic; this test script mirrors its flow for validation. |
| **`mnaas_move_files_from_staging_genric.sh`** | Populates `${staging_dir}/${processname}/data` with inbound files before this script runs. |
| **`mnaas_generic_recon_loading.sh`** | May consume the raw table populated by this script for reconciliation. |
| **Hadoop/HDFS** | Destination for Parquet files (`hdfs_parquet_location`). |
| **Hive / Impala** | Temporary and raw tables are created/updated; `msck repair` and `refresh` commands are issued. |
| **Java utilities** (`ParquetConverterUtility`, `RawTableLoading`) | Perform format conversion and bulk loading. |
| **Scheduler (cron / Oozie / Airflow)** | Triggers the script with appropriate arguments; the script checks the PID stored in the status file to avoid overlapping runs. |
| **Monitoring / Alerting** | Failure path sends email to `Cloudera.Support@tatacommunications.com` and creates an SDP ticket via `mailx`. |
| **Backup storage** | Configured via `${backuplocation}` – likely a mounted NFS or HDFS backup area. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID check may be bypassed if the status file is corrupted or the previous process crashes. | Duplicate processing, data corruption. | Implement a lock file (`flock`) in addition to PID check; ensure the status file is atomically updated. |
| **Missing or malformed configuration** (property files, process‑specific `.properties`). | Script aborts early, silent failures. | Validate existence of all sourced files at start; fail fast with clear error messages. |
| **File format mismatch** – `sed`/`awk` assumes CSV with commas; a different delimiter breaks downstream loading. | Data load failures, silent row loss. | Parameterize delimiter; add a pre‑check that validates a sample line. |
| **Java conversion failures** (OOM, classpath issues). | No Parquet files → downstream steps fail. | Capture Java exit code, log stdout/stderr, and retry with a smaller batch size. |
| **HDFS copy or Hive repair failures** – network hiccups. | Incomplete data in Hive. | Add retry loops with exponential back‑off for `hadoop fs -copyFromLocal` and `msck repair`. |
| **Hard‑coded paths** (e.g., `/app/hadoop_users/...`). | Breaks on environment changes. | Move all absolute paths to the central property files. |
| **Unimplemented functions** (`MNAAS_rm_header_from_files`, `MNAAS_move_files_to_another_temp_dir`). | Script will exit with “command not found”. | Ensure these functions are sourced from a shared library or inline them. |
| **Log file growth** – `set -x` plus verbose logging can fill disk. | Disk exhaustion, job abort. | Rotate logs daily; limit `set -x` to debug mode via an env flag. |

---

## 6. Running & Debugging the Script  

### 6.1 Typical Invocation  

```bash
# Example: load the KYC feed for the daily run
./mnaas_parquet_data_load_test.sh \
    -p KYC_FEED \
    -r daily \
    -s SEM \
    -f "*.txt"
```

* `-p` – logical process name (must match a directory under `${staging_dir}` and a `.properties` file).  
* `-r` – recurring identifier (e.g., `daily`, `hourly`).  
* `-s` – extension for the accompanying SEM file (defaults to `SEM`).  
* `-f` – glob pattern for source files.

### 6.2 Debug Steps  

1. **Check the log** – `${logdir}/mnaas_tbl_Load_${processname}.log_YYYY-MM-DD.Test`.  
2. **Inspect the status file** – `${daily_load_processesfilestatus}` to see the current flag and PID.  
3. **Validate environment variables** – `echo $staging_dir $logdir $backuplocation` before running.  
4. **Run with `bash -x`** (already enabled) or temporarily set `set -e` to abort on first error.  
5. **Force a specific step** – edit the flag in the status file to the desired numeric value (e.g., `3`) and re‑run; the script will resume from that step.  
6. **Manual Java test** – execute the Parquet conversion command directly on a sample file to verify classpath and JAR availability.  

---

## 7. External Configurations & Variables  

| File / Variable | Purpose |
|-----------------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop` | Global MNAAS environment (paths, Hadoop/Hive hosts, default directories). |
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed_tbl_Load.properties` | Process‑specific settings for the KYC feed (e.g., table names, query file). |
| `${staging_dir}/${processname}/config/${processname}.properties` | Overrides / additional parameters for the given process (e.g., `No_of_files_to_process`, validation regexes). |
| `daily_load_raw_process_status.lst` | Persistent status file used for flagging, PID tracking, and job‑run metadata. |
| `logdir` | Base directory for log files (defined in the global property file). |
| `backuplocation` | Root of the backup hierarchy where validated files are copied. |
| `hdfs_parquet_location` | HDFS target directory for Parquet files. |
| `temptable[]`, `rawtable[]`, `rejecttable[]` | Associative arrays mapping `processname` → Hive table names (populated by the sourced property files). |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath` | Java classpath and main JAR location for the loader utilities. |
| `Dname_MNAAS_Insert_Daily_KYC_tbl` | Name passed to the Java loader for identification. |
| `MNAAS_Daily_KYC_Feed_Load_LogPath` | Path for the Java loader’s internal logs. |
| `HIVE_HOST`, `IMPALAD_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Hive/Impala. |
| `SDP_ticket_*` variables (`SDP_ticket_to_email`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`) | Email routing for failure notifications. |

---

## 8. Suggested Improvements (TODO)

1. **Consolidate shared functions** – Move `MNAAS_rm_header_from_files` and `MNAAS_move_files_to_another_temp_dir` into a common library (e.g., `mnaas_common.sh`) and source it at the top. This eliminates “command not found” failures and makes the test script identical to production.  

2. **Replace ad‑hoc flag handling with a state machine** – Use a small JSON/YAML status file and a Bash `case` statement that maps flag values to function names. This improves readability, makes adding new steps easier, and reduces the long `if/elif` cascade.  

--- 

*End of documentation.*