**File:** `move-mediation-scripts\bin\MNAAS_Daily_KYC_Feed_tbl_loading.sh`  

---

## 1. High‑Level Summary
This Bash script implements the end‑to‑end daily load of KYC (Know‑Your‑Customer) feed files into the MNAAS data‑warehouse. It validates incoming CSV files, moves them through staging, backup, de‑duplication, and header‑removal steps, copies the cleaned data to HDFS, invokes Java loaders to populate a temporary Hive/Impala table, and finally merges the data into the production KYC snapshot table. The script maintains a process‑status file to allow safe restart/recovery and sends an SDP ticket + email on failure.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_files_pre_validation** | *Pre‑validation*: clears previous intermediate files, checks file naming pattern (`*_HOURLY_*.csv`), verifies text format, separates good, rejected, empty, and header‑only files; updates process‑status flag to **1**. |
| **MNAAS_cp_files_to_backup** | Copies all validated intermediate files to a daily backup directory; updates flag to **2**. |
| **MNAAS_rm_header_from_files** | Prepends the source filename as a column, strips header rows (`iccid`), appends cleaned rows to an incremental “dups” CSV; updates flag to **3**. |
| **MNAAS_remove_duplicates_in_the_files** | Merges all incremental files, removes duplicate rows (awk `!a[$0]++`), writes a consolidated snapshot file; updates flag to **4**. |
| **MNAAS_load_files_into_temp_table** | Uploads the de‑duplicated CSVs to HDFS, then runs a Java class (`Load_nonpart_table`) to bulk‑load into a temporary Hive table; refreshes the Impala metadata; updates flag to **5**. |
| **MNAAS_insert_into_kyc_snapshot_table** | Executes a second Java class (`RawTableLoading`) that moves data from the temp table into the final KYC snapshot table, cleans up intermediate directories, refreshes Impala; updates flag to **6** and terminates with success. |
| **terminateCron_successful_completion** | Resets status flags to *0* (Success), records run time, logs completion, exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends an SDP ticket + email to the MOVE support team (via `mailx`) if not already sent; updates status file to indicate ticket created. |

*Note:* The script also contains a commented‑out `MNAAS_save_partitions` placeholder.

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Daily_KYC_Feed_tbl_Load.properties` – defines all path variables, HDFS locations, DB names, Java class/jar names, email addresses, process‑status file, log file, etc. |
| **External Services** | - Hadoop HDFS (`hadoop fs` commands) <br> - Hive/Impala (`impala-shell`) <br> - Java runtime (custom JARs) <br> - System logger (`logger`) <br> - Mail system (`mailx`) |
| **Input Files** | - Raw KYC CSV files in `$MNAASMainStagingDirDaily_afr_seq_check/$Daily_KYC_Feed_Daily_Usage_ext` (limited by `$No_of_files_to_process`) <br> - Corresponding SHA256 checksum files (`*.sha256`) |
| **Intermediate Files** | - Validation‑status list (`$MNAASInterFilePath_Daily_KYC_Feed_Details/*`) <br> - Empty‑file list (`$MNNAS_Daily_KYC_OnlyHeaderFile`) <br> - Incremental duplicate files (`$Daily_Increment_KYC_Feed_Dups/*`) <br> - Consolidated snapshot (`$KYC_OneTimeSnapshot`) |
| **Outputs** | - Populated Hive/Impala tables (`kyc_daily_inter_tblname`, `kyc_iccid_wise_country_tblname`) <br> - Backup copy of validated files (`$Daily_KYC_BackupDir/`) <br> - Process‑status file (updated flags, timestamps, PID) <br> - Log file (`$MNAAS_Daily_KYC_Feed_Load_LogPath`) |
| **Side Effects** | - Moves/re‑moves files across staging, rejected, empty, backup directories <br> - Deletes intermediate files after successful load <br> - May trigger external ticketing/email on failure |

---

## 4. Integration Points & Connectivity  

| Component | How the script interacts |
|-----------|--------------------------|
| **Previous scripts** | The script is the “daily” counterpart of `MNAAS_Daily_KYC_Feed_Loading.sh` (the wrapper that may invoke this table‑loading script). It expects the same process‑status file and log path. |
| **Down‑stream consumers** | Any downstream analytics or reporting jobs that query the KYC snapshot Hive/Impala tables. |
| **Up‑stream feed producers** | External systems that drop hourly KYC CSV files into the staging directory (`$MNAASMainStagingDirDaily_afr_seq_check`). |
| **Java loaders** | `Load_nonpart_table` and `RawTableLoading` JARs (located via `$CLASSPATHVAR`, `$MNAAS_Main_JarPath`). These must be present and compatible with the Hive schema. |
| **Hadoop / Hive / Impala** | The script uses HDFS paths (`$MNAAS_Daily_Rawtablesload_KYC_PathName`), Hive DB (`$dbname`), and Impala hosts (`$IMPALAD_HOST`). Proper Kerberos tickets / permissions are required. |
| **Monitoring / Alerting** | Process‑status file is read by external watchdogs; SDP ticket/email notifies the MOVE support team. |
| **Cron / Scheduler** | Typically invoked by a daily cron job; the script guards against concurrent runs via PID stored in the status file. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID / false “already running”** | Job may be skipped unnecessarily. | Implement a PID age check (e.g., if PID > 24 h, treat as stale) and clean the status file on startup. |
| **Incorrect file naming pattern** | Valid files may be rejected. | Centralise the regex in the properties file; add a “dry‑run” mode that logs mismatches without moving files. |
| **HDFS / Hive permission failures** | Load step aborts, causing data loss. | Pre‑run a permission check script; alert on non‑zero exit from `hadoop fs -test -d`. |
| **Java loader crashes** | Partial data may be loaded, leaving tables inconsistent. | Wrap Java calls in a retry loop; use transactional Hive/Impala inserts where possible; keep a backup of raw CSVs until final success. |
| **Duplicate removal logic** | `awk '!a[$0]++'` may drop legitimate rows if they are identical but belong to different sources. | Include source filename as part of the dedup key (already done in `MNAAS_rm_header_from_files`). Verify with data‑quality team. |
| **Mailx/SDP ticket flood** | Repeated failures generate many tickets. | Add a throttling flag in the status file (already present) and ensure ticket is sent only once per run. |
| **Large file volume exceeding memory** | `cat` and `awk` on huge files may exhaust resources. | Process files in streaming mode (e.g., `awk` directly on HDFS) or split large files before dedup. |

---

## 6. Typical Execution / Debugging Steps  

1. **Run Manually (debug mode)**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files
   . MNAAS_Daily_KYC_Feed_tbl_Load.properties   # load env vars
   cd /path/to/move-mediation-scripts/bin
   bash -x MNAAS_Daily_KYC_Feed_tbl_loading.sh   # -x prints each command
   ```
2. **Check Process‑Status File**  
   ```bash
   cat $MNAAS_Daily_KYC_Feed_Load_ProcessStatusFileName
   ```
   Verify `MNAAS_Daily_ProcessStatusFlag` and `MNAAS_Script_Process_Id`.  

3. **Inspect Log**  
   ```bash
   tail -f $MNAAS_Daily_KYC_Feed_Load_LogPath
   ```
   Look for “successfully completed” messages or the point of failure.  

4. **Validate HDFS Upload**  
   ```bash
   hadoop fs -ls $MNAAS_Daily_Rawtablesload_KYC_PathName
   ```
5. **Confirm Hive/Impala Tables**  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM $dbname.$kyc_daily_inter_tblname;"
   impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM $dbname.$kyc_iccid_wise_country_tblname;"
   ```
6. **Re‑run from a Specific Step**  
   Edit the status file to set `MNAAS_Daily_ProcessStatusFlag` to the desired step (e.g., `3` to start at header removal) and re‑execute the script.  

7. **Failure Investigation**  
   - Check exit codes of the failing command (`echo $?`).  
   - Review the corresponding Java JAR logs (often under `$MNAAS_Daily_KYC_Feed_Load_LogPath`).  
   - Verify that the checksum files (`*.sha256`) exist for each processed CSV.  

---

## 7. External Configurations & Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_Daily_KYC_Feed_Load_LogPath` | Path to the script’s log file. |
| `MNAAS_Daily_KYC_Feed_Load_ProcessStatusFileName` | Central status/flag file used for restart logic and monitoring. |
| `MNAASMainStagingDirDaily_afr_seq_check` / `Daily_KYC_Feed_Daily_Usage_ext` | Source directory & file‑extension pattern for incoming KYC CSVs. |
| `MNAASInterFilePath_Daily_KYC_Feed_Details` | Working directory for validated files. |
| `MNAASRejectedFilePath` | Destination for malformed or non‑text files. |
| `EmptyFileDir` | Holds empty or header‑only files. |
| `Daily_KYC_BackupDir` | Backup location for validated files before load. |
| `Daily_Increment_KYC_Feed_Dups` | Holds per‑file de‑duplicated CSVs before final merge. |
| `KYC_OneTimeSnapshot` | Consolidated snapshot file (deduped). |
| `MNAAS_Daily_Rawtablesload_KYC_PathName` | HDFS directory where the final CSVs are copied for Hive load. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath and JAR locations for the loader programs. |
| `Load_nonpart_table`, `RawTableLoading` | Java class names invoked for temp‑table load and final insert. |
| `dbname`, `kyc_daily_inter_tblname`, `kyc_iccid_wise_country_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM` | Email addresses used when generating an SDP ticket on failure. |
| `No_of_files_to_process` | Upper limit on how many files to pick per run. |
| `MNAAS_Script_Process_Id` | PID stored for concurrency guard. |

*All of the above are defined in the sourced properties file; the script assumes they exist and are correctly set.*

---

## 8. Suggested Improvements (TODO)

1. **Modularise & Unit‑Test Validation Logic**  
   Extract the file‑name and format checks into a separate script or function library with unit tests (e.g., using `bats`). This will simplify maintenance and allow automated regression testing of naming conventions.

2. **Add Idempotent Checkpointing**  
   Persist a checksum (e.g., SHA256) of each processed file in a “processed‑files” ledger. Before moving a file to backup or loading, verify it hasn’t been processed already. This protects against accidental re‑runs caused by stale PID or manual restarts.  

---