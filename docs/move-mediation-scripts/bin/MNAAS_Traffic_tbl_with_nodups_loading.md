**MNAAS_Traffic_tbl_with_nodups_loading.sh – High‑Level Documentation**  

---

### 1. Purpose (one‑paragraph summary)  
This script orchestrates the daily “no‑duplicates” load of the **traffic_details_raw_daily_with_no_dups** table used by the MNAAS mediation pipeline. It sequentially (a) removes stale Hive/Impala partitions, (b) loads intermediate data into a temporary table, (c) merges the data into the final no‑dup table, and (d) runs a downstream SMS‑only processing step. Each stage updates a shared *process‑status* file, logs progress, and on failure triggers an email/SLA ticket via SDP. The script is designed to be idempotent across runs and is typically invoked by a nightly cron after upstream KYC feed loading completes.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **MNAAS_remove_partitions_from_the_table** | Sets status flag = 1, checks that the partition‑drop Java job is not already running, executes `drop_partitions_in_table` to delete old daily partitions, refreshes the Impala table, logs success/failure. |
| **MNAAS_Traffic_tbl_with_nodups_inter_table_loading** | Sets flag = 2, runs KYC feed loading script (`mnaas_move_files_from_staging_genric.sh`) as a prerequisite, then launches the Java class that loads data into a *temporary* “intermediate” table. |
| **MNAAS_Traffic_tbl_with_nodups_table_loading** | Sets flag = 3, runs the Java class that moves data from the intermediate table into the final no‑dup traffic table, refreshes Impala metadata, logs outcome. |
| **MNAAS_Traffic_tbl_with_nodups_sms_only** | Sets flag = 4, executes a separate shell script that extracts/loads SMS‑only records, logs outcome. |
| **terminateCron_successful_completion** | Resets status flags to *Success* (flag = 0, job_status=Success), clears the “email sent” flag, writes final log entries and exits 0. |
| **terminateCron_Unsuccessful_completion** | Writes failure status, calls `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Updates the status file to Failure, checks whether an SDP ticket/email has already been sent, and if not sends a formatted email (via `mail`/`mailx`) to the support mailbox and the Move‑Dev team. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Traffic_tbl_with_nodups_loading.properties` – defines log path, status‑file path, class names, JAR locations, DB hosts, ports, email recipients, etc. |
| **Environment variables** | `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`, `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, plus any variables referenced in the properties file (e.g., `$Dname_MNAAS_drop_partitions_Traffic_tbl_with_nodups_loading_tbl`). |
| **External services** | • Hive/Impala (metadata refresh via `impala-shell`).<br>• Java runtime (executing custom JAR classes).<br>• KYC feed staging area (accessed by `mnaas_move_files_from_staging_genric.sh`).<br>• SMTP server (used by `mail`/`mailx`). |
| **File system** | • Log file (`$MNAAS_Traffic_tbl_with_nodups_loadingLogPath`).<br>• Process‑status file (`$MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName`).<br>• Partition list file (`$MNAAS_Traffic_Partitions_Daily_Uniq_FileName`). |
| **Outputs** | • Updated Hive/Impala tables (`traffic_details_raw_daily_with_no_dups`).<br>• Log entries (standard output + error redirected to log).<br>• Process‑status file reflecting current flag, job status, PID, and email‑sent flag.<br>• Optional SDP ticket email on failure. |
| **Side‑effects** | • May delete/replace Hive partitions.<br>• Triggers downstream jobs that consume the no‑dup table.<br>• Sends email notifications and creates SDP tickets. |
| **Assumptions** | • All referenced JARs/classes are present and compatible with the current Hadoop/Impala version.<br>• The status file is writable and uniquely identifies this script’s run.<br>• No other process will manually edit the status file while the script runs.<br>• The KYC feed script completes within the allocated window. |

---

### 4. Integration Points (how this script connects to the rest of the system)  

| Connected Component | Interaction |
|---------------------|-------------|
| **MNAAS_Traffic_tbl_Load_driver.sh** (previous step) | Loads raw traffic data into a staging table; this script expects that data to be present before it runs. |
| **KYC Feed Loader** (`mnaas_move_files_from_staging_genric.sh`) | Executed inside `MNAAS_Traffic_tbl_with_nodups_inter_table_loading`; provides required KYC dimension data for aggregation. |
| **Java Partition‑Drop Class** (`drop_partitions_in_table`) | Removes old daily partitions; called from `MNAAS_remove_partitions_from_the_table`. |
| **Java Inter‑Table Load Class** (`$MNAAS_Traffic_tbl_with_nodups_inter_loading_load_classname`) | Populates a temporary table; invoked in `MNAAS_Traffic_tbl_with_nodups_inter_table_loading`. |
| **Java Final Load Class** (`$MNAAS_Traffic_tbl_with_nodups_loading_load_classname`) | Merges intermediate data into the final no‑dup table; invoked in `MNAAS_Traffic_tbl_with_nodups_table_loading`. |
| **SMS‑Only Processing Script** (`$MNAAS_traffic_no_dups_sms_only_ScriptName`) | Runs after the final load to generate SMS‑only aggregates. |
| **Impala Metastore** (`impala-shell -q "<refresh>"`) | Refreshes table metadata after partition changes or data loads. |
| **Monitoring / Alerting** | Status file is read by external dashboards; email/SDP ticket alerts are consumed by the support team. |
| **Cron Scheduler** | Typically invoked nightly via a crontab entry that sets the appropriate environment (e.g., `PATH`, `JAVA_HOME`). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID / false‑positive “already running”** | Script may skip a legitimate run, causing data lag. | Add a lock‑file with timestamp; if lock older than X hours, ignore and proceed. |
| **Java job failure (non‑zero exit)** | Partial data load, downstream jobs see inconsistent tables. | Verify exit codes, capture Java stdout/stderr to log, and abort with clear error code. |
| **Impala refresh failure** | New partitions not visible to downstream queries. | Retry `impala-shell` a configurable number of times; alert if still failing. |
| **KYC feed script failure** | Aggregations may miss KYC dimensions. | Separate KYC feed health check before starting main script; send early alert if missing files. |
| **Email/SDP ticket not sent** | Operations team unaware of failure. | Log the result of `mail`/`mailx`; if non‑zero, write to a fallback log and raise a system‑level alarm. |
| **Insufficient disk space for logs** | Log rotation may stop, causing script to hang on write. | Rotate logs daily, monitor disk usage, and fail fast with a clear message. |
| **Permission issues on status file** | Script cannot update flags, leading to stuck state. | Ensure the script runs under a dedicated service account with proper ACLs; validate at start. |

---

### 6. Execution & Debugging Guide  

1. **Prerequisites**  
   - Verify that the properties file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Traffic_tbl_with_nodups_loading.properties` is present and readable.  
   - Ensure the service account has write permission on the log and status files.  
   - Confirm Java, Hive, Impala, and mail utilities are in the `$PATH`.  

2. **Running the script**  
   ```bash
   # As the designated service user
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_Traffic_tbl_with_nodups_loading.sh
   ```
   - The script already runs with `set -x`; you will see each command echoed to the log.  
   - Exit code `0` = success, `1` = failure (also logged).  

3. **Manual flag manipulation (for testing)**  
   - The status file contains lines like `MNAAS_Daily_ProcessStatusFlag=2`.  
   - Edit the file to set a specific flag and re‑run to start from a particular stage.  

4. **Debugging steps**  
   - **Check the log**: `$MNAAS_Traffic_tbl_with_nodups_loadingLogPath`.  
   - **Validate Java class execution**: Run the Java command manually with `-verbose:class` to see class loading.  
   - **Impala refresh**: Execute the refresh query directly via `impala-shell` to verify connectivity.  
   - **PID lock**: If the script reports “Process/Jar is already running”, inspect the PID stored in the status file and verify with `ps -p <PID>`.  

5. **Post‑run verification**  
   - Query Hive/Impala: `SELECT COUNT(*) FROM traffic_details_raw_daily_with_no_dups;`  
   - Ensure no duplicate rows (e.g., using a `GROUP BY` on primary keys).  

---

### 7. External Configuration & Environment Variables  

| Variable (populated in properties file) | Meaning / Usage |
|------------------------------------------|-----------------|
| `MNAAS_Traffic_tbl_with_nodups_loadingLogPath` | Path to the script’s log file (stderr redirected). |
| `MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName` | Shared status/lock file containing flags, PID, job status, email‑sent flag. |
| `MNAAS_Traffic_Partitions_Daily_Uniq_FileName` | File listing daily partition identifiers to be dropped. |
| `traffic_details_raw_daily_with_no_dups_tblname_refresh` | Impala `REFRESH` statement string for the target table. |
| `Dname_MNAAS_drop_partitions_Traffic_tbl_with_nodups_loading_tbl` | Identifier used to guard against concurrent Java partition‑drop jobs. |
| `MNAAS_Main_JarPath`, `Generic_Jar_Names` | Locations of the JAR files containing the Java classes invoked. |
| `MNAAS_traffic_no_dups_sms_only_ScriptName` | Path to the SMS‑only processing script. |
| `MNAAS_Daily_KYC_Feed_tbl_Loading_Script` (commented) | Legacy KYC loading script (now replaced by generic staging script). |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email routing parameters for failure notifications. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Hosts/ports for Impala and Hive connections. |
| `MNAAS_traffic_aggr_adhoc_ScriptName` | Human‑readable name used in log/email subjects. |

---

### 8. Suggested Improvements (TODO)  

1. **Replace PID‑in‑status‑file lock with a robust lockfile** – Create a lock file under `/var/lock/mnaas/traffic_nodups.lock` that includes a timestamp; on start, acquire the lock atomically (`flock`) and release on exit. This avoids stale PID issues and simplifies concurrency control.  

2. **Centralise status handling** – Extract the flag‑update, logging, and email‑trigger logic into a shared library (e.g., `mnaas_status.sh`) so that all MNAAS scripts use a single, tested implementation, reducing duplication and the risk of inconsistent status files.  

--- 

*End of documentation.*