**High‑Level Documentation – `move-mediation-scripts/bin/product_status_report.sh`**  

---

### 1. Purpose (one‑paragraph summary)  
`product_status_report.sh` is the daily “eLUX Product Status” reporting job. It orchestrates the loading of several intermediate Hive tables (`elux_prod_status_sim_*`) into final reporting tables, extracts a consolidated CSV report via Impala, and publishes the file to an SFTP landing zone (with a backup copy and gzip compression). The script also updates a shared *process‑status* file used by the broader MOVE mediation framework, logs progress to a central log, and raises an SDP ticket via email if any step fails.

---

### 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **export_elux_report_pyspark** | Runs a Spark‑Python job (`$MNAAS_Product_Status_Report_Pyfile`) to generate a report in HDFS, copies it to SFTP, and triggers an Impala refresh. (Currently disabled – kept for legacy use.) |
| **export_elux_report_script** | *Main* reporting path: truncates and repopulates four Hive tables, runs an Impala `SELECT …` to produce a CSV, copies the file to a temporary location, backs it up, moves it to the final SFTP directory, sets permissions, gzips it, and logs success/failure. |
| **terminateCron_successful_completion** | Writes *Success* flags into the shared status file, logs completion timestamps, and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_on_reject_triggering_step`, and exits with code 1. |
| **email_on_reject_triggering_step** | Sends an SDP ticket‑style email (via `mailx`) to the support mailbox, indicating which script failed and when. |
| **Main program (bottom of file)** | Prevents concurrent runs by checking the PID stored in the status file, writes the current PID, reads the process‑flag, decides which export routine to run, and finally calls the appropriate termination routine. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration files** | `MNAAS_Daily_Product_Status_Report.properties` (provides all `$MNAAS_*` variables). |
| **Environment variables** | `setparameter` (evaluated before sourcing the properties), `IMPALAD_HOST`, `SDP_ticket_from_email`, `SDP_ticket_cc_email`. |
| **External services / systems** | • Hadoop YARN (Spark job)  <br>• HDFS (intermediate report storage)  <br>• Hive (table truncation / insert)  <br>• Impala (SQL query & refresh)  <br>• SFTP/FS path (`$MNAAS_Product_Status_Report_sftp_path*`)  <br>• Syslog (`logger`)  <br>• Mail server (`mailx`). |
| **Input data** | Source Hive tables: `mnaas.elux_prod_status_sim_det`, `…_sim_inv`, `…_sim_traffic`, `…_sim_top_up`. |
| **Primary output** | CSV file `eLUX_HOL_01_ProductStatus_<timestamp>.csv.gz` placed in `$MNAAS_Product_Status_Report_sftp_path`. A backup copy is stored in `$MNAAS_Product_Status_Report_back_path`. |
| **Side‑effects** | • Updates the shared process‑status file (`$MNAAS_Daily_Product_Status_Report_ProcessStatusFileName`). <br>• Writes to the daily log (`$MNAAS_Daily_Product_Status_Report_logpath`). <br>• May trigger an Impala table refresh (`$MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh`). |
| **Assumptions** | • All Hive tables exist and are populated by upstream ingestion jobs. <br>• Impala daemon reachable at `$IMPALAD_HOST`. <br>• SFTP directory is writable by the script user. <br>• `mailx` is configured to send external mail. <br>• The status file format is `key=value` lines (as used by other MOVE scripts). |

---

### 4. Integration Points (how it connects to other scripts/components)  

| Connected component | Interaction |
|---------------------|-------------|
| **Up‑stream ingestion scripts** (e.g., `mnaas_tbl_load.sh`, `mnaas_vaz_table_aggregation.sh`) | Populate the `elux_prod_status_sim_*` Hive tables that this script later truncates/loads. |
| **Process‑status file** (`$MNAAS_Daily_Product_Status_Report_ProcessStatusFileName`) | Shared with the MOVE scheduler and other daily jobs to coordinate execution order and detect failures. |
| **Impala refresh query** (`$MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh`) | Likely used by downstream reporting dashboards; this script triggers it after the Spark version (if enabled). |
| **SFTP landing zone** (`$MNAAS_Product_Status_Report_sftp_path`) | Consumed by external reporting tools or partner systems that pull the CSV. |
| **Backup path** (`$MNAAS_Product_Status_Report_back_path`) | Retained for audit/recovery; may be accessed by archival jobs. |
| **Cron / scheduler** | The script is invoked by a cron entry (or an orchestrator like Oozie/Airflow) that expects the script to exit with 0 on success and non‑zero on failure. |
| **SDP ticketing** | Failure emails are routed to the internal SDP ticketing system (`insdp@tatacommunications.com`). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID check may be bypassed if the status file is corrupted or the previous process hung. | Duplicate loads, file overwrites, resource contention. | Add a lockfile with `flock` and enforce a timeout; monitor the PID’s actual process state more robustly. |
| **Hive/Impala query failures** – No explicit error capture beyond exit code; partial tables may remain truncated. | Data loss / incomplete report. | Wrap each Hive statement in a transaction‑style check; if any step fails, roll back (re‑load from backup) or abort before truncation. |
| **SFTP write permission issues** – `chmod 777` is overly permissive and may mask permission problems. | Security exposure, silent failures. | Use stricter permissions (e.g., 750) and verify write success before proceeding. |
| **Hard‑coded email recipients** – Change in support address requires script edit. | Missed alerts. | Externalize email recipients in the properties file. |
| **Missing environment variables** – `setparameter` evaluation may fail silently. | Script aborts early, no clear log. | Validate required variables at start and abort with a clear error message. |
| **Large data volume** – Spark job (if re‑enabled) may exceed cluster resources. | Job OOM / long runtimes. | Parameterise executor counts via properties; add resource‑usage monitoring. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **1. Verify environment** | Ensure `$MNAAS_Daily_Product_Status_Report_ProcessStatusFileName` and all `$MNAAS_*` variables are defined (source the properties file manually). |
| **2. Dry‑run** | `bash -x product_status_report.sh` – the `set -x` already prints each command; watch the log for the PID check and flag values. |
| **3. Force a specific path** | Export `MNAAS_FlagValue=0` before execution to guarantee the script runs the `export_elux_report_script` branch. |
| **4. Check logs** | Tail `$MNAAS_Daily_Product_Status_Report_logpath` while the job runs. |
| **5. Verify output** | After success, confirm the gzipped CSV exists in `$MNAAS_Product_Status_Report_sftp_path` and a copy in the backup directory. |
| **6. Simulate failure** | Introduce a syntax error in one Hive statement or make the SFTP directory read‑only; observe that the script logs the failure, sends the SDP email, and exits with code 1. |
| **7. PID/Lock inspection** | `cat $MNAAS_Daily_Product_Status_Report_ProcessStatusFileName` to view stored PID; `ps -fp <pid>` to confirm the process is still running. |

---

### 7. External Config / Environment References  

| Variable | Source | Usage |
|----------|--------|-------|
| `MNAAS_Daily_Product_Status_Report_ProcessStatusFileName` | properties file | Stores flags, PID, and status strings; read/written throughout the script. |
| `MNAAS_Daily_Product_Status_Report_logpath` | properties file | Central log file for `logger` output. |
| `MNAAS_Product_Status_Report_hdfs_path` | properties file | Input path for Spark job (if used). |
| `MNAAS_Product_Status_Report_sftp_path` / `_temp` / `_back_path` | properties file | Destination directories for the CSV (final, temp, backup). |
| `MNAAS_Product_Status_Report_Pyfile` | properties file | Path to the Spark‑Python script (currently commented out). |
| `IMPALAD_HOST` | environment / properties | Hostname for `impala-shell` connections. |
| `SDP_ticket_from_email`, `SDP_ticket_cc_email` | environment | Email headers for the failure notification. |
| `setparameter` | external wrapper / calling environment | Evaluated before sourcing the properties file; likely contains `export` statements for the above variables. |

---

### 8. Suggested TODO / Improvements  

1. **Replace the ad‑hoc PID check with a robust lock mechanism** – use `flock` on a lockfile and store the lock’s PID; automatically release on script exit (trap).  
2. **Externalize all email‑related values** (recipients, subject template) into the properties file and add validation of required variables at script start.  

--- 

*End of documentation.*