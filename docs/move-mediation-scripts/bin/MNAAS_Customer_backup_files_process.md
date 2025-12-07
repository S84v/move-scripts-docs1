**MNAAS_Customer_backup_files_process.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_Customer_backup_files_process.sh` orchestrates the end‑to‑end handling of daily backup files generated per‑customer (traffic, tolling, actives, activation, etc.). It scans configured backup directories, extracts file‑level metadata, writes a flat‑file manifest, loads that manifest into a temporary Hive table, merges the data into a partitioned production Hive/Impala table, compresses and copies the original backup files to a secondary edge node via SCP, and finally cleans up the local copies. The script maintains a process‑status file that records the current step (flags 1‑5) and supports resumable execution if a previous run failed part‑way.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **`MNAAS_backup_files_to_process`** | - Sets flag 1 (processing). <br>- Iterates over all configured `file_type` and `customer` entries. <br>- For each matching backup file (limited by `No_of_files_to_process`) extracts metadata (customer, timestamps, record count, size, checksum name, etc.) and appends a delimited line to the manifest file (`$MNAAS_Customer_backup_file_process_filename`). |
| **`load_backup_files_records_to_temp_table`** | - Flag 2. <br>- Removes any previous HDFS staging data, copies the manifest to HDFS, truncates the temporary Hive table (`$customers_files_inter_record_count_tblname`) and loads the manifest into it. |
| **`insert_data_to_mainTable`** | - Flag 3. <br>- Inserts data from the temporary table into the partitioned production table (`$customers_files_records_count_tblname`) using dynamic partitions. <br>- Refreshes the Impala metadata for the target table. |
| **`customer_SCP_backup_files_to_edgenode2`** | - Flag 4. <br>- Gzips all backup files per customer/type, creates the destination directory on the second edge node (`$edge2node`), and copies the `.gz` files via `scp`. |
| **`customer_remove_backup_files`** | - Flag 5. <br>- Deletes the gzipped files from the local backup directories after successful transfer. |
| **`terminateCron_successful_completion`** | - Resets flag 0, writes success status, timestamps, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | - Logs failure, triggers email/SDP ticket (`email_and_SDP_ticket_triggering_step`), and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | - Sends a templated email to the support mailbox and marks the ticket‑creation flag to avoid duplicate alerts. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration file** | `MNAAS_Customer_backup_files_process.properties` – defines all paths, arrays (`MNAAS_backup_files_process_file_pattern`, `MNAAS_backup_files_process_backup_dir`, `MNAAS_backup_files_edgenode2_backup_dir`), filenames, delimiters, DB names, table names, log locations, etc. |
| **Environment variables** | `ENV_MODE` (PROD/DEV), `IMPALAD_HOST`, `edge2node`, `SDP_ticket_from_email`, `SDP_Receipient_List`. |
| **Process‑status file** | `$MNAAS_Customer_backup_files_process_ProcessStatusFileName` – holds flags, job status, timestamps, PID, etc.; read/written throughout the run. |
| **Manifest file (output)** | `$MNAAS_Customer_backup_file_process_filename` – a delimited text file containing one line per processed backup file (metadata). |
| **HDFS staging path** | `$MNAAS_Customer_backup_filename_hdfs_customer/*` – temporary location for the manifest before Hive load. |
| **Hive tables** | - Temporary: `$dbname.$customers_files_inter_record_count_tblname` (truncated each run). <br>- Production: `$dbname.$customers_files_records_count_tblname` (partitioned by `file_date`). |
| **External services** | - Hadoop HDFS (`hadoop fs` commands). <br>- Hive CLI (`hive -e`). <br>- Impala (`impala-shell`). <br>- Remote edge node (SSH/SCP). <br>- System logger (`logger`). <br>- Mail system (`mailx`). |
| **Side effects** | - Creates/updates status file and log file. <br>- Writes manifest to local FS and HDFS. <br>- Loads data into Hive/Impala tables. <br>- Transfers compressed backup files to a remote node. <br>- Deletes local compressed files after successful transfer. |
| **Assumptions** | - All directories and HDFS paths exist and are writable. <br>- Hive/Impala services are reachable. <br>- SSH keys/passwordless access to `$edge2node` is configured. <br>- The property file correctly defines associative arrays for each customer and file type. <br>- Sufficient disk space for temporary gzipping. |

---

### 4. Interaction with Other Scripts / Components  

| Connected Component | How this script interacts |
|---------------------|---------------------------|
| **MNAAS_*_tables_loading.sh** (or similar) | Consumes the production Hive table `$customers_files_records_count_tblname` populated by this script for downstream reporting/billing. |
| **MNAAS_Billing_Export_*.sh** | May read the same Hive tables to generate billing extracts; thus data freshness depends on successful completion of this script. |
| **Cron scheduler** | Typically invoked via a daily cron entry; the status‑file flag logic allows the job to resume if a previous run stopped at flag 2‑5. |
| **Monitoring/Alerting** | The status file is read by external health‑check scripts; SDP ticket creation integrates with the telecom’s incident‑management platform. |
| **Edge node 2** | Receives the gzipped backup files; downstream processes on that node (e.g., archival or further ETL) rely on the successful copy. |
| **Other “backup” scripts** (e.g., `MNAAS_CDR_Active_feed_customer_secs_mapping_table.sh`) | May generate the source backup files that this script later processes. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale or corrupted property file** | Wrong paths / missing arrays → script aborts or processes wrong files. | Validate the property file at start (e.g., check required variables, array lengths). |
| **HDFS / Hive load failure** | Data not persisted → downstream billing fails. | Add retry logic for `hadoop fs -copyFromLocal` and Hive `LOAD DATA`; monitor exit codes and alert early. |
| **SCP transfer failure** | Backup files not archived → data loss risk. | Verify checksum after copy, keep a copy until remote confirmation, and implement exponential back‑off retries. |
| **Concurrent executions** | Two instances may corrupt the status file or duplicate processing. | The PID check already prevents this; ensure the status file is on a shared FS with atomic writes. |
| **Disk space exhaustion during gzip** | Script stops mid‑run, leaving partially compressed files. | Pre‑check free space (`df`) before gzip; clean up old archives regularly. |
| **Missing Impala host or authentication failure** | Refresh command fails, causing flag 3 to be considered failed. | Test connectivity to `$IMPALAD_HOST` before running the insert step; fallback to Hive only if Impala unavailable. |
| **Email/SDP ticket spam** | Repeated failures may flood the ticketing system. | The `MNAAS_email_sdp_created` flag prevents duplicate tickets; ensure it is reset on successful run. |

---

### 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Customer_backup_files_process.properties` is present and readable.  
   - Verify environment variables (`ENV_MODE`, `IMPALAD_HOST`, `edge2node`, `SDP_ticket_from_email`, `SDP_Receipient_List`).  
   - Confirm SSH key‑based access to `$edge2node`.  

2. **Typical Invocation** (via cron or manually)  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Customer_backup_files_process.sh
   ```

3. **Manual Debugging Steps**  
   - **Check current flag**: `grep MNAAS_CDR_Backup_ProcessStatusFlag $MNAAS_Customer_backup_files_process_ProcessStatusFileName`.  
   - **Force start from a specific step**: edit the flag to the desired value (e.g., `=2`) and re‑run.  
   - **Enable verbose tracing**: the script already runs with `set -x`; you can also redirect stdout/stderr to a temporary log for deeper inspection.  
   - **Validate manifest**: after `MNAAS_backup_files_to_process` finishes, inspect `$MNAAS_Customer_backup_file_process_filename` for correct delimited rows.  
   - **Test HDFS copy**: `hadoop fs -ls $MNAAS_Customer_backup_filename_hdfs_customer/`.  
   - **Run Hive load manually**: copy the `hive -e "LOAD DATA …"` command from the log and execute it step‑by‑step.  

4. **Log Locations**  
   - Primary log: `$MNAAS_Customer_backup_files_process_logpath`.  
   - Process‑status file: `$MNAAS_Customer_backup_files_process_ProcessStatusFileName`.  

5. **Exit Codes**  
   - `0` – successful completion (status file set to Success).  
   - `1` – failure; an SDP ticket/email has been generated.

---

### 7. External Config / Files Referenced  

| File / Variable | Role |
|-----------------|------|
| `MNAAS_Customer_backup_files_process.properties` | Central configuration – defines all directory paths, file patterns, delimiters, DB/table names, log paths, email settings, and associative arrays for customers and file types. |
| `$MNAAS_Customer_backup_file_process_filename` | Manifest file generated by the script; later uploaded to HDFS. |
| `$MNAAS_Customer_backup_files_process_ProcessStatusFileName` | Persistent status/flag file used for resumability and monitoring. |
| `$MNAAS_Customer_backup_files_process_logpath` | Log file written via `logger -s`. |
| `$MNAAS_Customer_backup_filename_hdfs_customer` | HDFS staging directory for the manifest. |
| `$edge2node` & `$MNAAS_backup_files_edgenode2_backup_dir` | Remote host and target directory for backup file transfer. |
| `$IMPALAD_HOST` | Impala daemon used for metadata refresh. |
| `$SDP_ticket_from_email`, `$SDP_Receipient_List` | Email parameters for incident ticket creation. |

---

### 8. Suggested Improvements (TODO)

1. **Add Robust Parameter Validation** – Implement a `validate_config` function that checks existence/readability of all directories, HDFS paths, and required variables before any processing begins. Exit early with a clear error if any check fails.

2. **Introduce Retry/Back‑off Logic for External Calls** – Wrap `hadoop fs`, `hive`, `impala-shell`, and `scp` commands in a helper that retries up to *n* times with exponential back‑off, logging each attempt. This will reduce transient network/hadoop failures causing full job aborts.  

(Additional enhancements such as moving from Bash to a more maintainable language like Python, or using a workflow engine (e.g., Oozie/Airflow), are beyond the immediate scope but worth considering for long‑term reliability.)