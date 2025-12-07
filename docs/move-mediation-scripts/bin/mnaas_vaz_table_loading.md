**File:** `move-mediation-scripts\bin\mnaas_vaz_table_loading.sh`  

---

## 1. High‑Level Summary
This Bash orchestration script loads VAZ (Voice‑Access‑Zone) reconciliation data into the MNAAS data‑warehouse. It pulls CSV files from a staging location via `rsync`, moves them into an intermediate HDFS landing area, loads them into a temporary Hive table, merges the data into a daily partitioned reporting table, backs up the source files, and finally records the distinct partition dates for downstream processes. Execution state is persisted in a property file that acts as a lightweight lock and status tracker, and optional success/failure email notifications are sent to the IPX team.

---

## 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **start()** | Sources the property file, redirects `stderr` to the script log, and stores the command‑line arguments for later use. |
| **mnaas_generic_vaz_enrich()** | *Step 1* – Copies new CSV files from the remote staging server (`$SOURCE` → `$DESTINATION`) via `rsync`; moves processed files on the remote host to a backup folder; moves local landing CSVs to the intermediate HDFS staging directory (`$mnaasinterfilepath_daily_generic_vaz`). |
| **mnaas_generic_vaz_temp_table_load()** | *Step 2* – Creates the HDFS staging directory, copies the intermediate CSVs into HDFS, then runs a Java JAR (`$Load_nonpart_table`) to bulk‑load the data into a non‑partitioned temporary Hive table (`$generic_vaz_inter_tblname`). Refreshes the table in Impala. |
| **mnaas_generic_vaz_table_load()** | *Step 3* – Executes a Hive `INSERT OVERWRITE` partition‑insert query (read from the properties file) via `beeline` to merge the temporary data into the daily reporting table (`$generic_vaz_report`). Refreshes the table in Impala. |
| **mnaas_copy_vaz_files_to_backup()** | *Step 4* – Moves the processed CSVs from the intermediate local directory to a backup directory (`$generic_backupdir`) and gzips them. |
| **MNAAS_save_partitions()** | *Step 5* – Generates a list of distinct partition dates from the reporting table, writes them to an HDFS directory (`$HDFS_VAZ_PARTITION_DATA`), and concatenates the result into a local file (`$MNAAS_VAZ_Partitions_Hourly_FileName`). |
| **terminateCron_successful_completion()** | Writes a “Success” status, clears the lock flag, optionally sends a success email, logs completion, and exits `0`. |
| **terminateCron_Unsuccessful_completion()** | Writes a “Failure” status, triggers failure email/SDP ticket, logs, and exits `1`. |
| **email_success_triggering_step()** | Sends a success email with the generated report attached (only once per run). |
| **email_and_SDP_ticket_triggering_step()** | Sends a failure email and creates an SDP ticket (once per failure). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Command‑line arguments** | `-p <processname>` (identifier), `-f <filelike>` (pattern), `-t <filetype>` (type), `-e <emailtype>` (`yes`/`no`), optional `-s` startdate, `-l` lastdate (currently unused). |
| **External configuration** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaas_vaz_table_loading.properties` – defines all paths, HDFS locations, Hive/Impala hosts, JAR names, log filenames, email lists, etc. |
| **External services** | • Remote staging host `vaz@10.171.102.16` (rsync/ssh)  <br>• HDFS cluster (via `hdfs dfs`, `hadoop fs`)  <br>• Hive/Impala (via `beeline`, `impala-shell`)  <br>• Java runtime (for bulk‑load JAR)  <br>• Mail system (`mailx`, `mail`) for notifications. |
| **Primary inputs** | CSV files matching `$filelike` that appear in `$SOURCE` (remote) and are landed in `$mnaas_vaz_data_landingdir`. |
| **Primary outputs** | 1. Populated Hive tables (`$generic_vaz_inter_tblname`, `$generic_vaz_report`). <br>2. Log file (`$mnaas_vaz_table_loadinglogname`). <br>3. Backup gzip files in `$generic_backupdir`. <br>4. Partition list file (`$MNAAS_VAZ_Partitions_Hourly_FileName`). |
| **Side effects** | • Updates the status property file (`$mnaas_vaz_table_loadprocessstatusfilename`) with flags, PID, timestamps. <br>• Moves remote log files to a backup folder on the staging host. <br>• May trigger external SDP ticket creation on failure. |
| **Assumptions** | • All required environment variables are defined in the sourced properties file. <br>• The remote host is reachable via password‑less SSH. <br>• HDFS, Hive, Impala services are up and the user has required permissions. <br>• The Java JAR (`$Load_nonpart_table`) is present and compatible with the Hive schema. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | How it ties in |
|---------------------|----------------|
| **`MNAAS_move_files_from_staging.sh`** | Likely responsible for initially placing the CSV files on the remote staging host (`$SOURCE`). This script later pulls them via `rsync`. |
| **`MNAAS_Weekly_KYC_Feed_Loading.sh`** | May generate additional reference data that populates lookup tables used by the VAZ reporting Hive queries. |
| **`api_med_subs_activity_loading.sh`** | Could feed subscriber activity data that is joined with VAZ reconciliation data in downstream analytics. |
| **Property file (`mnaas_vaz_table_loading.properties`)** | Central configuration shared across all VAZ‑related scripts; defines common paths, DB names, and email lists. |
| **HDFS directories (`$HDFS_VAZ_PARTITION_DATA`, `$mnaasinterfilepath_daily_generic_vaz`)** | Shared staging area used by other ingestion pipelines that need to read the same partition list. |
| **Email/SDP notification system** | Other scripts also use the same `email_and_SDP_ticket_triggering_step` logic, ensuring consistent alerting across the platform. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale lock / PID mismatch** – If the script crashes without clearing the PID, subsequent runs will be blocked. | Production data may stop loading. | Implement a watchdog that clears the lock after a configurable timeout; log PID start time and compare with current time. |
| **Partial rsync failure** – Network glitch may copy only a subset of files, leaving the rest in staging. | Incomplete data in Hive, downstream reporting errors. | Verify file counts before and after rsync; add checksum validation; move files to a “failed” folder on remote host if mismatch. |
| **Java bulk‑load JAR failure** – Schema changes or missing JAR cause load to abort. | Temporary table not populated → downstream steps fail. | Add pre‑run validation of JAR existence and version; capture JAR exit code and log detailed error. |
| **Hive/Impala query failure** – Partition insert may fail due to syntax or permission issues. | Data not refreshed, stale reports. | Store the exact Hive query in a separate file for easier debugging; enable Hive query logging. |
| **Email/SDP flood** – Repeated failures could generate many tickets/emails. | Alert fatigue. | Add a throttling mechanism (e.g., only send one ticket per hour per process). |
| **Hard‑coded IP/hostnames** – Changes in infrastructure break connectivity. | Script stops at rsync/ssh. | Externalize hostnames/IPs into the properties file and document change‑control procedures. |

---

## 6. Running / Debugging the Script  

1. **Preparation**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaas_vaz_table_loading.properties` is present and contains correct values for all variables referenced in the script.  
   - Verify password‑less SSH to `vaz@10.171.102.16` and that the remote directory `$SOURCE` exists.  
   - Confirm HDFS, Hive, Impala services are reachable from the host where the script runs.  

2. **Typical invocation (operator)**  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/mnaas_vaz_table_loading.sh \
       -p VAZ_RECON \
       -f "*.csv" \
       -t VAZ \
       -e yes
   ```
   - `-e yes` enables success/failure email notifications.  

3. **Debugging steps**  
   - The script starts with `set -x`; all commands are echoed to stdout, which can be captured in the log file defined by `$mnaas_vaz_table_loadinglogname`.  
   - After a run, inspect the status file (`$mnaas_vaz_table_loadprocessstatusfilename`) to see the final flag (`MNAAS_Daily_ProcessStatusFlag`).  
   - If the script exits early, check the log for the line where `terminateCron_Unsuccessful_completion` was called.  
   - To isolate a step, comment out later function calls in the `if` block and re‑run; verify intermediate outputs (e.g., HDFS directory contents, Hive table row counts).  

4. **Manual recovery**  
   - If the lock PID is stale, edit the status file and set `MNAAS_Script_Process_Id=0` and `MNAAS_Daily_ProcessStatusFlag=0`.  
   - Re‑run the script; it will start from the enrichment step.  

---

## 7. External Config / Environment Dependencies  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `mnaas_vaz_table_loadinglogname` | Path to the script’s log file (stderr redirected here). |
| `mnaas_vaz_table_loadprocessstatusfilename` | Persistent status/lock file storing flags, PID, timestamps, email‑sent flag. |
| `SOURCE`, `DESTINATION` | Remote staging directory (source) and local landing directory (destination) for rsync. |
| `mnaas_vaz_data_landingdir` | Local directory where raw CSVs are initially placed. |
| `mnaasinterfilepath_daily_generic_vaz` | Intermediate local directory before HDFS copy. |
| `mnaas_vaz_load_generic_pathname` | HDFS staging path for bulk load. |
| `generic_vaz_inter_tblname` | Temporary Hive table name used by the Java loader. |
| `generic_vaz_report` | Final daily reporting Hive table (partitioned). |
| `generic_backupdir` | Local backup directory for processed CSVs. |
| `Load_nonpart_table` | Java class (main) invoked for bulk loading. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath and JAR list. |
| `dbname`, `generic_vaz_inter_colname`, `generic_vaz_inter_tblname`, `generic_vaz_report` | Hive database and column/table names used in queries. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Impala and Hive. |
| `ccList`, `GTPMailId` | Email recipients for notifications. |
| `HDFS_VAZ_PARTITION_DATA`, `MNAAS_VAZ_Partitions_Hourly_FileName` | HDFS directory and local file for storing distinct partition dates. |

If any of these variables are missing or malformed, the script will fail early (e.g., `sed` commands will not find the target file).

---

## 8. Suggested Improvements (TODO)

1. **Lock Management Refactor** – Replace the ad‑hoc PID flag in the properties file with a proper lock file (e.g., `/var/run/mnaas_vaz.lock`) that includes a timestamp and supports automatic expiration. This reduces the risk of stale locks and simplifies status checks.

2. **Idempotent Rsync & Checksum Validation** – After `rsync`, compute a checksum (e.g., `md5sum`) of each transferred file and compare it with a checksum generated on the source host. Move only verified files to the intermediate directory; otherwise, log and alert. This ensures data integrity across unreliable networks.