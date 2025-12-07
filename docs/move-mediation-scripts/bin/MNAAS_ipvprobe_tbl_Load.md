**File:** `move-mediation-scripts/bin/MNAAS_ipvprobe_tbl_Load.sh`

---

## 1. High‑Level Summary
`MNAAS_ipvprobe_tbl_Load.sh` is the daily ETL driver that ingests raw IPV‑Probe usage CSV files produced by the Mediation layer, validates and normalises them, removes duplicates, and loads the cleaned data into Hive/Impala tables (`ipvprobe_daily_inter_tblname`, `ipvprobe_daily_tblname`, `ipvprobe_daily_reject_tblname`). The script is orchestrated via a flag‑based state machine stored in a shared *process‑status* file, allowing it to resume from the last successful step after a failure or a manual restart. All major steps are logged, backed‑up, and any fatal error triggers an SDP ticket and email notification.

---

## 2. Core Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | - Sets flag = 1 in status file.<br>- Clears previous intermediate files.<br>- Lists incoming files, validates naming pattern and that they are plain‑text.<br>- Moves good files to `$MNAASInterFilePath_Daily_IPVProbe_Details`, rejects malformed/empty/header‑only files to `$MNAASRejectedFilePath` or `$EmptyFileDir`. |
| **MNAAS_cp_files_to_backup** | - Flag = 2.<br>- Copies the validated intermediate files to `$Daily_IPVProbe_BackupDir` for audit/recovery. |
| **MNAAS_rm_header_from_files** | - Flag = 3.<br>- Strips the first line (CSV header) from each intermediate file.<br>- Moves the associated `.sem` semaphore file to the backup dir. |
| **MNAAS_append_filename_to_start_of_records** | - Flag = 4.<br>- Prefixes every record with the source filename (`filename;record`). |
| **MNAAS_move_files_to_another_temp_dir** | - Flag = 5.<br>- Copies the prefixed files to a temporary “with‑dups” directory (`$MNASS_IPVProbe_Inter_removedups_withdups_filepath`). |
| **MNAAS_remove_duplicates_in_the_files** | - Flag = 6.<br>- Removes duplicate lines per file using `awk '!a[$0]++'` and writes to a “without‑dups” directory (`$MNASS_IPVProbe_Inter_removedups_withoutdups_filepath`). |
| **MNAAS_load_files_into_temp_table** | - Flag = 7.<br>- Removes any previous HDFS load path, copies the de‑duplicated files to HDFS, then runs a Java loader (`Load_nonpart_table`) to populate the *intermediate* Hive table (`$ipvprobe_daily_inter_tblname`).<br>- Refreshes the Impala metadata. |
| **MNAAS_insert_into_raw_table** | - Flag = 8.<br>- Executes a Java inserter (`Insert_Part_Daily_table`) to move data from the intermediate table to the final raw table (`$ipvprobe_daily_tblname`).<br>- Cleans up local intermediate files and refreshes Impala. |
| **MNAAS_insert_into_reject_table** | - Flag = 9.<br>- Runs a Java inserter (`Insert_Part_Daily_Reject_table`) to move rejected rows (e.g., validation failures) into a dedicated reject Hive table (`$ipvprobe_daily_reject_tblname`). |
| **terminateCron_successful_completion** | - Resets flag = 0, marks job status *Success*, writes run‑time, logs completion, and exits 0. |
| **terminateCron_Unsuccessful_completion** | - Logs failure, calls `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | - Sets job status *Failure*.<br>- Sends a templated email via `mailx` to the SDP ticketing address and CCs the Move dev team (only once per run). |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ipvprobe_tbl_Load.properties`. Expected variables (non‑exhaustive):<br>• `$MNAAS_Daily_ipvprobe_Load_Raw_LogPath` – log file path.<br>• `$MNAAS_Daily_ipvprobe_Load_Raw_ProcessStatusFileName` – status‑flag file.<br>• `$MNAASMainStagingDirDaily_afr_seq_check` – inbound staging directory.<br>• `$IPVPROBE_Daily_Usage_ext` – file extension pattern (e.g., `*.csv`).<br>• `$MNAASInterFilePath_Daily_IPVProbe_Details`, `$MNAASRejectedFilePath`, `$EmptyFileDir`, `$Daily_IPVProbe_BackupDir` – various working/backup dirs.<br>• `$MNASS_IPVProbe_Inter_removedups_withdups_filepath`, `$MNASS_IPVProbe_Inter_removedups_withoutdups_filepath` – temp dup‑handling dirs.<br>• `$MNAAS_Daily_Rawtablesload_ipvprobe_PathName` – HDFS target dir.<br>• `$CLASSPATHVAR`, `$Generic_Jar_Names`, `$MNAAS_Main_JarPath`, `$Load_nonpart_table`, `$Insert_Part_Daily_table`, `$Insert_Part_Daily_Reject_table` – Java classpath & jar names.<br>• Hive/Impala connection vars (`$HIVE_HOST`, `$HIVE_JDBC_PORT`, `$IMPALAD_HOST`).<br>• Email/SDP vars (`$SDP_ticket_from_email`, `$SDP_ticket_to_email`, `$MOVE_DEV_TEAM`). |
| **External Services** | - Hadoop HDFS (`hadoop fs` commands).<br>- Hive/Impala (metadata refresh via `impala-shell`).<br>- Java loader/inserter JARs (run as separate processes).<br>- System logger (`logger`).<br>- Mail subsystem (`mailx`). |
| **File System Side‑Effects** | - Moves/renames raw CSV files from staging to intermediate, backup, empty, reject directories.<br>- Generates checksum/empty‑header lists (`$MNNAS_IPVProbe_OnlyHeaderFile`).<br>- Writes/updates the process‑status file (flags, PID, timestamps).<br>- Creates HDFS files under `$MNAAS_Daily_Rawtablesload_ipvprobe_PathName`. |
| **Outputs** | - Populated Hive tables: `$ipvprobe_daily_inter_tblname`, `$ipvprobe_daily_tblname`, `$ipvprobe_daily_reject_tblname`.<br>- Log file (`$MNAAS_Daily_ipvprobe_Load_Raw_LogPath`).<br>- Backup copies of source files (`$Daily_IPVProbe_BackupDir`). |
| **Assumptions** | - All required directories exist and are writable by the script user.<br>- Incoming files follow the naming regex `^[a-zA-Z]+_[a-zA-Z]+_[a-zA-Z]+_[0-9]+_[0-9]+\.csv$`.<br>- Corresponding `.sem` semaphore files are present alongside each CSV.<br>- Java classpath and JARs are compatible with the Hadoop/Hive versions in use.<br>- Only one instance runs at a time (PID guard). |

---

## 4. Integration Points & Call Graph

| Component | Relationship |
|-----------|--------------|
| **MNAAS_ipvprobe_daily_recon_loading.sh** | Likely downstream consumer that runs reconciliation queries after the raw tables are populated. |
| **MNAAS_ipvprobe_monthly_recon_loading.sh** | Monthly counterpart that aggregates the daily tables populated by this script. |
| **MNAAS_ipvprobe_seq_check.sh** | Generates the `$MNAASMainStagingDirDaily_afr_seq_check` directory and may create the `.sem` files used for sequencing. |
| **Java Loader JARs** (`Load_nonpart_table`, `Insert_Part_Daily_table`, `Insert_Part_Daily_Reject_table`) | Executed from this script; they read the HDFS staging area and write to Hive. |
| **Hadoop / HDFS** | Provides the raw‑tables load path; other scripts may clean or archive this path. |
| **Impala / Hive Metastore** | Tables created/updated here are read by reporting, billing, and analytics pipelines. |
| **Process‑status file** (`$MNAAS_Daily_ipvprobe_Load_Raw_ProcessStatusFileName`) | Shared with other orchestration scripts (e.g., daily batch controller) to monitor progress and enable restart. |
| **Logging & Alerting** | `logger` writes to syslog; `mailx` creates SDP tickets that are consumed by the incident‑management system. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (PID guard failure) | Duplicate loads, data corruption | Verify the PID file is reliably cleared on abnormal exit; add a lockfile with `flock`. |
| **Missing or malformed configuration** | Script aborts early, silent failures | Validate all required variables after sourcing the properties file; fail fast with clear log messages. |
| **File naming mismatch** | Good files may be rejected unnecessarily | Keep the regex in a configurable property; add a “dry‑run” mode that only logs matches. |
| **Large duplicate‑removal step** (`awk '!a[$0]++'`) may exhaust memory for very large files | OOM, job failure | Switch to `sort -u` on HDFS or stream‑based deduplication; limit file size per batch. |
| **Java loader failures** (exit code non‑zero) | Partial data load, downstream jobs fail | Capture stdout/stderr of Java processes; implement retry logic with exponential back‑off. |
| **Hard‑coded paths** (e.g., `/app/hadoop_users/...`) | Breaks on environment changes | Externalise all paths into the properties file; add sanity checks at start. |
| **Email/SDP spam** (multiple tickets for same failure) | Alert fatigue | Ensure the `MNAAS_email_sdp_created` flag is atomically updated; consider a throttling window. |
| **No cleanup of HDFS temp dir on failure** | Stale files consume space | Add a cleanup step in `terminateCron_Unsuccessful_completion`. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file `MNAAS_ipvprobe_tbl_Load.properties` is present and all variables are exported.  
   - Verify that the staging directory (`$MNAASMainStagingDirDaily_afr_seq_check`) contains the expected CSV files and matching `.sem` files.  
   - Confirm Hadoop, Hive, Impala, Java, `mailx`, and `logger` are in the PATH.

2. **Manual Execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts/bin
   ./MNAAS_ipvprobe_tbl_Load.sh   # run as the same user that cron uses
   ```
   - The script will write to the log file defined by `$MNAAS_Daily_ipvprobe_Load_Raw_LogPath`.  
   - To force a fresh run, reset the flag in the status file:  
     ```bash
     sed -i 's/^MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=0/' $MNAAS_Daily_ipvprobe_Load_Raw_ProcessStatusFileName
     ```

3. **Debug Mode**  
   - The script already runs with `set -x` (trace). For more verbose output, tail the log while the job runs:  
     ```bash
     tail -f $MNAAS_Daily_ipvprobe_Load_Raw_LogPath
     ```

4. **Inspecting State**  
   - Current flag & PID:  
     ```bash
     grep -E 'MNAAS_Daily_ProcessStatusFlag|MNAAS_Script_Process_Id' $MNAAS_Daily_ipvprobe_Load_Raw_ProcessStatusFileName
     ```
   - List files in each working directory to verify movement.

5. **Common Failure Points & Checks**  
   - **Java process not starting**: `ps -ef | grep $Dname_MNAAS_Load_Daily_ipvprobe_tbl_temp` – ensure no stray processes.  
   - **HDFS permission**: `hadoop fs -ls $MNAAS_Daily_Rawtablesload_ipvprobe_PathName`.  
   - **Impala refresh errors**: capture the output of `impala-shell` (add `-B` for batch mode) and inspect.  

---

## 7. External Config / Environment Variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_Daily_ipvprobe_Load_Raw_LogPath` | Absolute path of the script’s log file. |
| `MNAAS_Daily_ipvprobe_Load_Raw_ProcessStatusFileName` | Shared status/flag file used for restartability and PID guard. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Directory where inbound IPV‑Probe CSV files land. |
| `IPVPROBE_Daily_Usage_ext` | File‑extension pattern (e.g., `*.csv`). |
| `MNAASInterFilePath_Daily_IPVProbe_Details` | Temp directory for validated files. |
| `MNAASRejectedFilePath` | Destination for rejected files. |
| `EmptyFileDir` | Destination for empty or header‑only files. |
| `Daily_IPVProbe_BackupDir` | Long‑term backup of raw files before transformation. |
| `MNASS_IPVProbe_Inter_removedups_withdups_filepath` / `MNASS_IPVProbe_Inter_removedups_withoutdups_filepath` | Temp dirs for duplicate handling. |
| `MNAAS_Daily_Rawtablesload_ipvprobe_PathName` | HDFS staging path for the load into the intermediate Hive table. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components for loader/inserter JARs. |
| `Load_nonpart_table`, `Insert_Part_Daily_table`, `Insert_Part_Daily_Reject_table` | Main class names invoked by Java. |
| `dbname`, `ipvprobe_daily_inter_tblname`, `ipvprobe_daily_tblname`, `ipvprobe_daily_reject_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM` | Email addresses used for failure notifications. |
| `MNAASDailyIPVProbeLoadScriptName` | Human‑readable script identifier used in logs/emails (set in properties). |

*If any of the above are missing or empty, the script will likely abort with a “command not found” or “cannot write to file” error.*

---

## 8. Suggested Improvements (TODO)

1. **Modularise Duplicate Removal** – Replace the in‑memory `awk '!a[$0]++'` with a Hadoop‑side `sort -u` or Spark job to handle very large files without risking OOM on the driver node.

2. **Centralised Locking Mechanism** – Implement a `flock`‑based lockfile (e.g., `/var/lock/MNAAS_ipvprobe_tbl_Load.lock`) instead of editing the status file for PID guarding. This provides atomic acquisition and automatic cleanup on process termination.  

---