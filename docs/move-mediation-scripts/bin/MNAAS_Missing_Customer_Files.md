**MNAAS_Missing_Customer_Files.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Missing_Customer_Files.sh` is a daily monitoring job that validates the completeness of customer‑specific data files ingested by the MNAAS mediation platform. For the previous day it compares “File Received” vs “File Generated” KPI counts in Impala, extracts any missing filenames per process and customer, builds a CSV summary and a plain‑text list, and emails the results to the development/operations team. The script also updates a shared process‑status file, logs its execution, and triggers an SDP ticket on failure to ensure visibility and downstream recovery.

---

### 2. Key Functions / Logical Blocks

| Function / Block | Responsibility |
|------------------|----------------|
| **`MNAAS_Missing_Customer_Files`** | Core workflow: set status flags, run KPI diff query, iterate over `process_list`/`customer_names` (and InfONova customers) to collect missing filenames, generate reports, send email, update status file on success. |
| **`terminateCron_successful_completion`** | Reset status flags, write success metadata (run time, PID) to the status file, log completion, exit 0. |
| **`terminateCron_Unsuccessful_completion`** | Log failure, invoke `email_and_SDP_ticket_triggering_step`, exit 1. |
| **`email_and_SDP_ticket_triggering_step`** | Mark job status as *Failure* in the status file, create an SDP ticket flag (once per run) and log the action. |
| **Main Program (bottom of file)** | Guard against concurrent runs using the PID stored in the status file, decide whether to start the job based on the `MNAAS_Daily_ProcessStatusFlag`, and invoke the core function followed by appropriate termination handling. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Missing_Customer_Files.properties` – defines paths, hostnames, arrays (`process_list`, `customer_names`, `infonova_customer_names`), email recipients, status‑file location, etc. |
| **Environment / External Services** | - **Impala** (`impala-shell -i $IMPALAD_HOST`) – runs SQL queries against `mnaas.kpi_measurement`, `mnaas.telena_file_record_count`, `mnaas.customers_files_records_count`.<br>- **Mail server** – `mailx` used to send reports.<br>- **File system** – reads/writes status file, temporary query outputs, CSV/TXT reports, log file. |
| **Inputs (runtime)** | - `filedate` – yesterday’s date (derived inside script).<br>- Process & customer arrays from the properties file.<br>- Existing status file (`$MNAAS_Missing_Customer_Files_ProcessStatusFilename`). |
| **Outputs** | - `$missing_files_report` – CSV with KPI diff per customer/process.<br>- `$query_output1` – plain‑text list of missing filenames (`customer_filename`).<br>- Log entries appended to `$MNAAS_Missing_Customer_Files_logpath`.<br>- Email with the two reports attached, sent to `$MOVE_DEV_TEAM` (CC list defined). |
| **Side Effects** | - Updates the shared status file (flags, PID, run time, job status, SDP‑ticket flag).<br>- May trigger an SDP ticket (external ticketing system) via the `email_and_SDP_ticket_triggering_step` routine.<br>- Alters file permissions (`chmod 777`) on generated reports. |
| **Assumptions** | - Impala host reachable and the required tables exist with expected schema.<br>- The properties file correctly defines all referenced variables.<br>- Mailx is configured and the recipient addresses are valid.<br>- The script runs with a user that has read/write/execute rights on all paths. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | How the Connection Occurs |
|---------------------|----------------------------|
| **Other MNAAS daily jobs** (e.g., `MNAAS_*_Load.sh`, `MNAAS_*_Check.sh`) | All share the same *process‑status* file (`$MNAAS_Missing_Customer_Files_ProcessStatusFilename`). Those jobs read the flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`) to decide whether to start or to wait. |
| **KPI Measurement Pipeline** | The script queries `mnaas.kpi_measurement` which is populated by upstream ingestion jobs (e.g., `MNAAS_Load_SOTA_Table.sh`). Missing‑file alerts feed back to those pipelines for corrective action. |
| **Ticketing / Alerting System** | `email_and_SDP_ticket_triggering_step` writes a flag (`MNAAS_email_sdp_created=Yes`) that is later consumed by an external SDP ticket automation script (not shown). |
| **Monitoring / Logging** | Uses `logger -s` to send messages to syslog; other monitoring tools (e.g., Splunk, Nagios) may ingest these logs. |
| **Cron Scheduler** | Typically invoked from a daily cron entry; the PID guard prevents overlapping runs, a pattern used across the suite of MNAAS scripts. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Impala query failure / timeout** | No report generated; downstream processes think data is complete. | Capture exit code of `impala-shell`; add retry logic; alert on non‑zero exit before proceeding to email step. |
| **Missing or malformed properties file** | Script aborts early, status file left in inconsistent state. | Validate required variables after sourcing; fail fast with clear log message. |
| **Concurrent execution (PID guard race)** | Duplicate runs could corrupt status file or generate duplicate tickets. | Use an atomic lock file (`flock`) instead of PID check; ensure lock removal on abnormal exit. |
| **Email delivery failure** | Stakeholders not notified; issue may go unnoticed. | Verify `mailx` return code; fallback to an alternative notification channel (e.g., Slack webhook). |
| **File permission changes (`chmod 777`)** | Overly permissive files may expose sensitive data. | Restrict to appropriate group/owner (`chmod 664`); audit after generation. |
| **Large customer/process lists** | Long runtime, possible memory pressure on the host. | Profile query execution time; consider batching or parallelizing per process. |
| **SDP ticket duplication** | Multiple tickets for the same failure. | Ensure the `MNAAS_email_sdp_created` flag is checked and persisted atomically. |

---

### 6. Typical Execution / Debugging Steps

1. **Run Manually**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files
   source MNAAS_Missing_Customer_Files.properties
   bash /path/to/MNAAS_Missing_Customer_Files.sh
   ```
2. **Enable Verbose Logging** (already set with `set -x`). Review `/path/to/logfile` defined by `$MNAAS_Missing_Customer_Files_logpath`.  
3. **Check Status File** after run:  
   ```bash
   cat $MNAAS_Missing_Customer_Files_ProcessStatusFilename
   ```
   Verify `MNAAS_job_status=success` and `MNAAS_email_sdp_created=No`.  
4. **Validate Reports** – ensure `$missing_files_report` (CSV) and `$query_output1` (TXT) exist and contain expected rows.  
5. **Inspect Email** – confirm receipt and attachment integrity.  
6. **Failure Simulation** – temporarily break the Impala host name or remove a required variable, re‑run, and verify that the script logs the failure, updates the status file to `Failure`, and sets the SDP ticket flag.  

---

### 7. External Configuration & Environment Variables

| Variable (populated in properties file) | Role |
|----------------------------------------|------|
| `MNAASConfPath` | Base directory for generated reports (`Missing_Customer_Files` sub‑folder). |
| `MNAAS_Missing_Customer_Files_logpath` | Path to the script’s log file. |
| `MNAAS_Missing_Customer_Files_ProcessStatusFilename` | Shared status/flag file used across MNAAS jobs. |
| `IMPALAD_HOST` | Impala daemon host for `impala-shell`. |
| `process_list` (array) | List of process names to check (e.g., `traffic`, `billing`). |
| `customer_names` (array) | List of regular customers to monitor. |
| `infonova_customer_names` (array) | List of InfONova customers (special handling). |
| `MNAAS_Missing_Customer_Files_Scriptname` | Script identifier stored in status file. |
| `MOVE_DEV_TEAM` | Primary email recipient list for the report. |
| `mail_from`, `mail_to`, `cc_List` | Email metadata used by `mailx`. |
| `query_output`, `query_output1` | Temporary files for intermediate query results (paths derived from `$MNAASConfPath`). |

If any of these are missing or point to non‑existent locations, the script will fail early; therefore the properties file must be audited whenever the environment changes.

---

### 8. Suggested Improvements (TODO)

1. **Replace PID‑file guard with `flock`** – ensures atomic lock acquisition and automatic release on script termination, eliminating race conditions.  
2. **Add robust error handling for `impala-shell`** – capture exit status, log the error, and optionally retry a configurable number of times before marking the job as failed.  

---