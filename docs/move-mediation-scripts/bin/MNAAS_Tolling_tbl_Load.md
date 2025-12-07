**File:** `move-mediation-scripts/bin/MNAAS_Tolling_tbl_Load.sh`  

---

## 1. Purpose (one‑paragraph summary)

`MNAAS_Tolling_tbl_Load.sh` is the daily “Tolling” data‑pipeline driver. It validates incoming tolling CSV files, backs them up, strips headers, de‑duplicates against the previous day’s payload, stages the cleaned files to HDFS, loads them into a temporary Hive/Impala table, merges the data into a raw tolling table and finally (optionally) runs an aggregation step. Throughout the run it updates a shared *process‑status* file, writes detailed logs, and on failure sends an email and creates an SDP ticket. The script is orchestrated by a cron scheduler and is part of the broader “Move” mediation suite that processes CDR‑type data for downstream analytics.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | Scans the staging directory for the newest `$No_of_files_to_process` files matching the tolling naming pattern, verifies they are readable text CSVs, moves good files to the *intermediate* folder and malformed/rejected files to a reject folder. Updates process‑status flag = 1. |
| **MNAAS_cp_files_to_backup** | Copies all intermediate files to a daily backup directory. Updates flag = 2. |
| **MNAAS_rm_header_from_files** | Removes the first line (header) from every intermediate CSV. Updates flag = 3. |
| **MNAAS_remove_dups_file** | Performs file‑level de‑duplication against the previous day’s processed file using `sort` + `comm`. Emits only new rows (prefixed with the filename) to a global diff file, updates the “previous processed” folder, and removes duplicate files. Updates flag = 4. |
| **MNAAS_mv_files_to_hdfs** | Clears the target HDFS raw‑load directory, then copies all remaining intermediate CSVs to `$MNAAS_Daily_Rawtablesload_Tolling_PathName`. Updates flag = 5. |
| **MNAAS_load_files_into_temp_table** | Executes a Java loader (`$Load_nonpart_table`) that reads the HDFS files into a temporary Hive/Impala table (`$tolling_daily_inter_tblname`). Refreshes the table via `impala-shell`. Updates flag = 6. |
| **MNAAS_insert_into_raw_table** | Executes a Java inserter (`$Insert_Part_Daily_table`) that merges the temporary table into the permanent raw tolling table (`$tolling_daily_tblname`). Refreshes via `impala-shell`. Updates flag = 7. |
| **MNAAS_insert_into_aggr_table** | (Currently commented out) Intended to run a Java job that aggregates raw data into a daily aggregation table (`$tolling_aggr_daily_tblname`). Would set flag = 8. |
| **terminateCron_successful_completion** | Resets status flags to *idle* (0), marks job as `Success`, writes final timestamps, logs completion and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email to the MOVE dev team and creates an SDP ticket (via `mailx`) if not already done. Updates status file to indicate ticket creation. |

---

## 3. Inputs, Outputs & Side Effects

### 3.1 Inputs
| Item | Source |
|------|--------|
| `$1` – `Tolling_extn` | Command‑line argument – file extension/pattern (e.g., `*.csv`). |
| `$2` – `MNASS_Prev_Processed_filepath_Tolling` | Command‑line argument – directory holding the previous day’s processed file. |
| **Properties file** `MNAAS_Tolling_tbl_Load.properties` | Sourced at start; defines all environment variables used throughout (paths, DB names, Hadoop/Impala hosts, Java class names, log locations, email lists, etc.). |
| **Process‑status file** `$MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFileName` | Read/write to coordinate restart/recovery. |
| **Staging directory** `$MNAASMainStagingDirDaily_afr_seq_check/$Tolling_extn` | Holds inbound raw toll‑files. |
| **Java JARs & CLASSPATH** | `$CLASSPATHVAR`, `$Generic_Jar_Names`, `$MNAAS_Main_JarPath`. |
| **Hadoop / Hive / Impala** | Cluster services reachable via `$nameNode`, `$HIVE_HOST`, `$IMPALAD_HOST`. |

### 3.2 Outputs
| Item | Destination |
|------|-------------|
| Validated & cleaned CSVs | `$MNAASInterFilePath_Daily_Tolling` (intermediate) → HDFS `$MNAAS_Daily_Rawtablesload_Tolling_PathName`. |
| Rejected files | `$MNAASRejectedFilePath`. |
| Empty files | `$EmptyFileDir`. |
| Backup copies | `$Daily_Tolling_BackupDir`. |
| Global diff file (new rows) | `$Tolling_diff_filepath_global` (later moved back to intermediate folder). |
| Hive/Impala tables | `tolling_daily_inter_tblname`, `tolling_daily_tblname` (raw), optional aggregation table. |
| Log file | `$MNAAS_DailyTollingLoadAggrLogPath`. |
| Process‑status file updates | `$MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFileName`. |
| Failure email & SDP ticket | Sent to `${ccList}`, `${GTPMailId}`, `insdp@tatacommunications.com`. |

### 3.3 Side Effects
* File system moves, copies, deletions.
* HDFS directory clean‑up and uploads.
* Execution of external Java processes (potentially heavy MapReduce/Tez jobs).
* Hive/Impala table refreshes.
* Email and ticket creation on failure.
* Process lock via PID stored in the status file.

### 3.4 Assumptions
* The properties file exists and defines **all** referenced variables.
* Hadoop, Hive, Impala, and Java runtime are correctly installed and reachable.
* Only one instance of the script runs at a time (PID lock enforced).
* Input files follow the naming regex `^[a-zA-Z]+_[0-9]+_[a-zA-Z]+_[0-9]+_[0-9]+\.csv$`.
* Sufficient disk space in staging, backup, and HDFS target directories.
* Mail utilities (`mail`, `mailx`) are configured for outbound SMTP.

---

## 4. Interaction with Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **MNAAS_Tolling_seq_check.sh** (previous script) | Generates the staging directory `$MNAASMainStagingDirDaily_afr_seq_check` and may set `$No_of_files_to_process`. This script consumes those files. |
| **MNAAS_Tolling_tbl_Load.properties** | Central configuration shared across the tolling pipeline (paths, DB names, class names). |
| **Java loader JARs** (`$Load_nonpart_table`, `$Insert_Part_Daily_table`) | Provide the actual bulk‑load logic into Hive/Impala. |
| **Process‑status file** (`$MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFileName`) | Shared with monitoring/alerting tools; other scripts read the flag to decide whether to start or resume. |
| **HDFS raw‑load path** (`$MNAAS_Daily_Rawtablesload_Tolling_PathName`) | Consumed later by downstream analytics jobs (e.g., daily reporting, billing). |
| **SDP ticketing system** | Triggered via `mailx` on failure; external incident‑management integration. |
| **Cron scheduler** | Typically invoked nightly; ensures only one instance runs (PID lock). |
| **Potential downstream aggregation scripts** (currently commented) | Would read the raw table and produce aggregated metrics for reporting. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID lock** – script crashes leaving PID in status file → subsequent runs aborted. | Data backlog, missed SLA. | Add a health‑check that validates the PID’s process is alive; if stale, clear the lock automatically. |
| **HDFS upload failure** (network, permission, full disk). | Incomplete data load, downstream jobs fail. | Verify HDFS free space before copy; capture Hadoop exit codes; retry with exponential back‑off; alert on failure. |
| **Java loader crashes** (OOM, schema mismatch). | Raw table not populated, data loss. | Pre‑run schema validation; monitor Java process memory; enforce JVM limits; capture logs to a separate file for analysis. |
| **Duplicate‑file detection logic** may incorrectly drop legitimate rows if file ordering changes. | Data loss. | Add checksum comparison or record‑level deduplication in Hive instead of file‑level; keep a audit trail of removed rows. |
| **Mail/SLA notification flood** if script repeatedly fails. | Alert fatigue. | Implement rate‑limiting on ticket creation; include a “snooze” flag in the status file. |
| **Hard‑coded sed edits** on the status file can corrupt it if the file format changes. | Monitoring blind spots. | Switch to a key‑value store (e.g., JSON or properties file) and use a parser instead of `sed`. |
| **Insufficient disk space** in backup or intermediate directories. | Job aborts mid‑process. | Add pre‑run disk‑usage checks; rotate old backups; alert when thresholds exceed. |

---

## 6. Running & Debugging the Script

### 6.1 Typical Invocation
```bash
# Example: process all *.csv files, previous‑day folder supplied
./MNAAS_Tolling_tbl_Load.sh "*.csv" /data/tolling/prev_processed
```
*The first argument is a glob or extension; the second is the directory that holds the previous day’s processed file.*

### 6.2 Prerequisite Checks
1. Verify the properties file exists and is readable:
   ```bash
   test -f /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Tolling_tbl_Load.properties
   ```
2. Ensure required directories exist (staging, backup, reject, empty, HDFS target).
3. Confirm Hadoop, Hive, Impala, Java, and mail utilities are on the `$PATH`.

### 6.3 Monitoring Execution
* The script runs with `set -x` – all commands are echoed to stdout; redirect to a log if needed:
  ```bash
  ./MNAAS_Tolling_tbl_Load.sh "*.csv" /data/tolling/prev_processed > /tmp/tolling_run_$(date +%Y%m%d%H%M).log 2>&1
  ```
* Real‑time tail of the main log:
  ```bash
  tail -f $MNAAS_DailyTollingLoadAggrLogPath
  ```
* Check the process‑status file for the current flag:
  ```bash
  grep MNAAS_Daily_ProcessStatusFlag $MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFileName
  ```

### 6.4 Debugging Tips
| Symptom | Likely Cause | Debug Action |
|---------|--------------|--------------|
| No files processed, script exits early | Staging dir empty or pattern mismatch | `ls -l $MNAASMainStagingDirDaily_afr_seq_check/$Tolling_extn` |
| Files end up in *rejected* folder | Header/format validation failed | Inspect a sample file; verify it matches the regex and is plain‑text CSV. |
| HDFS copy fails | Permissions or full HDFS directory | Run `hadoop fs -ls $MNAAS_Daily_Rawtablesload_Tolling_PathName` and `hadoop fs -du -s`. |
| Java loader returns non‑zero | Classpath or DB connection issue | Look at `$MNAAS_DailyTollingLoadAggrLogPath` for the Java stack trace; test connection to Hive via `beeline`. |
| No email on failure | `mailx` not configured or `$SDP_ticket_from_email` empty | Manually run the mail command with a test message. |

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Meaning | Typical Use |
|---------------------------|---------|-------------|
| `MNAAS_DailyTollingLoadAggrLogPath` | Path to the main log file. | `logger -s … >> $MNAAS_DailyTollingLoadAggrLogPath` |
| `MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFileName` | Shared status/flag file. | Updated via `sed` throughout the script. |
| `MNAASMainStagingDirDaily_afr_seq_check` | Root staging directory for inbound files. | Source of raw CSVs. |
| `Tolling_extn` (set from `$1`) | File‑extension/pattern for selection. | Used in `ls -tr $MNAASMainStagingDirDaily_afr_seq_check/$Tolling_extn`. |
| `MNAASInterFilePath_Daily_Tolling` | Intermediate folder for validated files. | Holds files before HDFS copy. |
| `MNAASRejectedFilePath` | Rejected‑file folder. | Files failing validation are moved here. |
| `MNAASBackupFilePath` / `Daily_Tolling_BackupDir` | Backup location for raw copies. | Populated by `MNAAS_cp_files_to_backup`. |
| `MNAAS_Daily_Rawtablesload_Tolling_PathName` | HDFS target directory. | `hadoop fs -copyFromLocal …`. |
| `Load_nonpart_table`, `Insert_Part_Daily_table` | Fully‑qualified Java class names for loading/inserting. | Executed via `java -cp …`. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components. | Required for the Java jobs. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Hive/Impala service endpoints. | Used by `impala-shell` refresh commands. |
| `MNAASDailyTollingAggregationScriptName` | Human‑readable script identifier for logs/emails. | Set in properties. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email routing parameters. | Used in failure notification. |
| `No_of_files_to_process` | Max number of files to pick per run. | Controls loop in `MNAAS_files_pre_validation`. |
| `EmptyFileDir` | Directory for zero‑byte files. | Clean‑up step. |
| `Tolling_diff_filepath`, `Tolling_diff_filepath_global` | Temporary diff files for new rows. | Used in de‑duplication. |
| `MNAAS_Dups_Handlingpath_Tolling` | Scratch folder for sorted files during duplicate check. | Holds intermediate sorted files. |
| `MNASS_Prev_Processed_filepath_Tolling` (argument) | Directory containing yesterday’s processed file. | Used for de‑duplication comparison. |

---

## 8. Suggested Improvements (TODO)

1. **Replace ad‑hoc `sed` status updates with a structured JSON/YAML status file** – easier to parse, less fragile, and supports future fields without regex‑based edits.  
2. **Add explicit argument validation and usage help** – currently the script assumes two positional arguments; a `--help` flag and checks for empty or malformed inputs would prevent accidental mis‑runs.  

*(Further enhancements could include moving to a proper logging