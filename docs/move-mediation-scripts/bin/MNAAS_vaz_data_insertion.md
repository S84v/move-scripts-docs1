**MNAAS_vaz_data_insertion.sh – High‑Level Documentation**

---

### 1. Summary
`MNAAS_vaz_data_insertion.sh` is the daily “VAZ” data‑load orchestrator for the Move‑Mediation platform. It pulls CSV files from a remote VAZ router (`vaz@10.171.102.16`) via `rsync`, stages them in HDFS, loads the data into a temporary Hive/Impala table, then merges it into the final raw VAZ table. The script maintains a process‑status file to survive restarts, prevents concurrent executions, and generates success/failure logs, SDP tickets, and email alerts.

---

### 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **MNAAS_export_files_into_linux** | Copies new VAZ CSV files from the remote router to the local staging directory (`$DESTINATION`) using `rsync`; moves processed remote files to a backup folder; updates process‑status flag to **1**. |
| **MNAAS_load_files_into_temp_table** | Uploads the staged CSV (`$MNAAS_Local_Input_Filename`) to HDFS (`$MNAAS_VAZ_HDFS_PATH`), runs a Java loader to populate a non‑partitioned temporary Hive table, then refreshes the corresponding Impala metadata. Updates process‑status flag to **2**. |
| **MNAAS_insert_into_raw_table** | Executes a Java job that inserts data from the temporary table into the final raw VAZ Hive table, compresses the original CSV backup, cleans HDFS, and refreshes Impala metadata. Updates process‑status flag to **3**. |
| **terminateCron_successful_completion** | Resets the status file to “idle” (`flag=0`, `job_status=Success`), records run time, writes final log entries and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | If an SDP ticket has not yet been created, sends a templated email to the support mailbox and marks the ticket‑created flag in the status file. |
| **Main driver block** | Checks the PID stored in the status file to avoid parallel runs, reads the current flag, and executes the appropriate steps (export → load → insert) based on the flag value. Handles empty‑file cases and final cleanup. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration (external)** | `MNAAS_VAZ_Data_Insertion.properties` – defines all variables referenced (`$SOURCE`, `$DESTINATION`, `$MNAAS_Local_Input_Filename`, `$VAZ_IP_ADDRESS`, `$VAZ_PORT_NUMBER`, `$VAZ_control_file`, `$Tablename`, `$CLASSPATHVAR`, `$Generic_Jar_Names`, `$Load_nonpart_table`, `$Insert_Weekly_table`, `$HIVE_HOST`, `$IMPALAD_HOST`, `$VAZ_weekly_inter_tblname`, `$VAZ_weekly_tblname_refresh`, `$MNAAS_Vaztable_Load_Raw_ProcessStatusFileName`, `$MNAAS_VAZ_Data_InsertionLogPath`, etc.). |
| **Environment** | Assumes Java, Hadoop CLI (`hdfs dfs`), Impala‑shell, `rsync`, `ssh`, `mailx`, and the required JARs are on the classpath. Runs under a Hadoop user account (`/app/hadoop_users/MNAAS`). |
| **Input files** | Remote CSV files located at `vaz@10.171.102.16:/logs/router/script/vaz_dm_cellIdlocation/*.csv`. Local staging file `$MNAAS_Local_Input_Filename` (single aggregated CSV). |
| **Output files** | Log file `$MNAAS_VAZ_Data_InsertionLogPath`. Process‑status file `$MNAAS_Vaztable_Load_Raw_ProcessStatusFileName`. Backup of processed CSVs in `$Daily_Telena_BackupDir` (gzipped). HDFS files under `$MNAAS_VAZ_HDFS_PATH`. |
| **Side effects** | - Remote files are moved to a backup directory after successful rsync.<br>- HDFS directories are populated and later cleared.<br>- Hive/Impala tables are refreshed.<br>- System logger entries (`logger -s`).<br>- SDP ticket email sent on failure (via `mailx`). |
| **Assumptions** | - Only one instance of the script runs at a time (enforced via PID in status file).<br>- The remote VAZ host is reachable via SSH key‑based auth for `rsync` and `ssh` commands.<br>- Java loader JARs accept the arguments as shown.<br>- Impala and Hive services are up and reachable (`$IMPALAD_HOST`, `$HIVE_HOST`).<br>- The status file exists and is writable. |

---

### 4. Integration with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **MNAAS_move_files_from_staging_ipvprobe.sh** (and similar staging scripts) | Those scripts move raw files from various sources into the generic staging area (`/Input_Data_Staging/MNAAS_DailyFiles/`). `MNAAS_vaz_data_insertion.sh` expects its own VAZ files to be present in `$DESTINATION` (populated by its own `rsync`). |
| **MNAAS_report_data_loading.sh** | After VAZ raw table is populated, downstream reporting jobs (e.g., daily aggregation scripts) read from the same Hive/Impala tables. |
| **MNAAS_table_statistics_calldate_aggr.sh** | May depend on the raw VAZ data for call‑date statistics; runs after this script completes successfully. |
| **Properties file** (`MNAAS_VAZ_Data_Insertion.properties`) | Shared across the Move‑Mediation suite; any change to variable names or paths must be reflected in all dependent scripts. |
| **Java loader JARs** (`vaz_mysql_connector.jar`, `Insert_Weekly_table`, `Load_nonpart_table`) | Executed by this script; other scripts may reuse the same JARs for different tables. |
| **SDP ticketing / email** | Uses the same mail template and ticket‑creation logic as other failure‑handling scripts (e.g., `MNAAS_ipvprobe_seq_check.sh`). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID check may be bypassed if the status file is corrupted or the PID is reused. | Duplicate loads, data duplication or table lock. | Add a lock file (`flock`) in addition to PID check; monitor lock file age and clean up stale locks. |
| **Remote rsync failure** – network glitch leaves files partially transferred. | Incomplete data, downstream job failures. | Verify file checksum after rsync (e.g., `rsync --checksum`) or compare file counts before/after. |
| **Java loader crash** – non‑zero exit not captured correctly. | Data not loaded, HDFS files left behind. | Wrap Java calls in a retry loop with exponential back‑off; ensure HDFS cleanup on failure. |
| **Impala refresh failure** – stale metadata leads to query errors. | Downstream reports return stale or empty results. | Capture Impala exit code; if non‑zero, retry refresh or raise alert immediately. |
| **Log / status file growth** – unlimited appends may fill disk. | Disk exhaustion, script aborts. | Rotate log files daily (logrotate) and truncate status file after each successful run. |
| **Hard‑coded remote host** (`10.171.102.16`) – single point of failure. | No data if host down. | Externalize host/IP in properties file; add a fallback host list. |
| **SDP ticket spam** – repeated failures may generate many tickets. | Alert fatigue. | Implement a throttling counter in the status file (e.g., only send ticket if last ticket > 1 h ago). |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Prerequisite** | Ensure the properties file `MNAAS_VAZ_Data_Insertion.properties` is present and all variables are correctly set. Verify SSH key access to `vaz@10.171.102.16`. |
| **Execute** | ```bash /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_vaz_data_insertion.sh``` (run as the MNAAS Hadoop user). |
| **Dry‑run / Verbose** | The script starts with `set -x`; it already prints each command. To capture full trace, redirect stdout/stderr: ```bash ... > /tmp/vaz_debug.log 2>&1``` |
| **Check status** | Review `$MNAAS_VAZ_Data_InsertionLogPath` for the latest run. Inspect `$MNAAS_Vaztable_Load_Raw_ProcessStatusFileName` for flag values and PID. |
| **Force re‑run** | If a previous run hung, delete the PID line in the status file or set `MNAAS_Daily_ProcessStatusFlag=0` manually, then re‑execute. |
| **Debug Java loader** | Add `-verbose:class` or increase Java logging inside the JARs; capture output in the same log file. |
| **Validate HDFS** | ```hdfs dfs -ls $MNAAS_VAZ_HDFS_PATH``` – confirm file presence and permissions. |
| **Validate Hive/Impala** | ```impala-shell -i $IMPALAD_HOST -q "SHOW TABLES LIKE '<temp_table>'; SELECT COUNT(*) FROM <raw_table>;"``` |
| **Simulate failure** | Rename a required variable in the properties file to trigger the failure path and verify that an SDP ticket email is sent. |

---

### 7. External Configuration & Dependencies  

| Item | Usage |
|------|-------|
| **`MNAAS_VAZ_Data_Insertion.properties`** | Provides all runtime variables (paths, hostnames, table names, JAR locations, flags). The script sources it with `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_VAZ_Data_Insertion.properties`. |
| **Environment variables** | `$PATH` must include Hadoop binaries (`hdfs`, `impala-shell`), Java, `rsync`, `ssh`, `mailx`. |
| **JARs** | `$VAZ_connector_jar`, `$MNAAS_Main_JarPath`, `$Generic_Jar_Names` – required for Java loading steps. |
| **Mail configuration** | `mailx` must be configured to reach `insdp@tatacommunications.com`; `$SDP_ticket_from_email` and `$MOVE_DEV_TEAM` are defined in the properties file. |
| **Log rotation** | Not handled in the script; external logrotate should manage `$MNAAS_VAZ_Data_InsertionLogPath`. |

---

### 8. Suggested TODO / Improvements  

1. **Add a robust locking mechanism** – replace the PID‑only check with `flock` on a lock file to guarantee single‑instance execution even if the status file becomes inconsistent.  
2. **Externalize remote host & credentials** – move `$SOURCE`, `$DESTINATION`, and the remote IP into the properties file (or a secure vault) and add a fallback host list to improve resiliency.  

--- 

*End of documentation.*