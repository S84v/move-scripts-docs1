**File:** `move-mediation-scripts/bin/customreportgenerator.sh`  

---

## 1. High‑Level Summary
`customreportgenerator.sh` is a Bash driver that produces customer‑specific custom reports for the MNAAS (Mobile Network Analytics & Assurance System) platform. It reads a property file to obtain configuration, parses command‑line arguments (`-r` report name, `-d` report date, `-s` customer SECS ID), and then invokes a Java class (`com.tcl.custom.report.ReportExporterMain`) to generate CSV files. When the special report name **ALL** is supplied, the script iterates over a comma‑separated list of report types and customer IDs defined in the property file, generating each combination. Successful runs are logged; failures trigger an error‑handling routine that may raise an SDP ticket and send an email when running in production.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `secsidwisereportgeneration()` | Core driver: decides between bulk (“ALL”) and single‑report mode, loops over reports/customers, launches the Java exporter, logs success/failure, and (commented) optional SCP transfer. |
| `terminateCron_successful_completion()` | Writes a success message to the log and exits with status 0. |
| `terminateCron_Unsuccessful_completion()` | Writes a failure message, optionally triggers email/SDP ticket (if `ENV_MODE=PROD`), and exits with status 1. |
| `email_and_SDP_ticket_triggering_step()` | Checks a process‑status flag file; if no ticket has been raised, composes and sends an email via `mailx` and logs the action. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Command‑line inputs** | `-r <reportname>` (e.g., `ALL` or a specific report), `-d <reportdate>` (YYYY‑MM‑DD), `-s <tcl_secs_id>` (customer SECS ID). |
| **Configuration file** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaascustomreport.properties` – supplies `logdir`, `reportexportpath`, `customreport`, `customerid`, `setparameter`, `MNAAS_Daily_report_traffic_no_dup_*` variables, `ENV_MODE`, etc. |
| **Environment variables** | Expected from the sourced property file: `logdir`, `reportexportpath`, `customreport`, `customerid`, `setparameter`, `MNAAS_Daily_report_traffic_no_dup_ProcessStatusFileName`, `MNAAS_Daily_report_traffic_no_dup_scriptName`, `MNAAS_Daily_report_traffic_no_dup_log_file_path`, `ENV_MODE`. |
| **External binaries** | `java` (with a large classpath pointing to Hive, Hadoop, and custom JARs), `mailx`. |
| **Outputs** | CSV files written to `${reportexportpath}/${rep}_${secs}_${reportdate}.csv` (or `${reportname}_${tcl_secs_id}_${reportdate}.csv`). |
| **Side effects** | - Appends diagnostic messages to `${logdir}/customreportgenerator.sh`. <br> - May send email and create an SDP ticket on failure (production only). <br> - (Commented) optional `scp` of the CSV to a staging host. |
| **Assumptions** | • Property file exists and defines all required variables.<br>• Java class `ReportExporterMain` is functional and reachable via the supplied JAR.<br>• The script runs on a node with Hadoop/Hive libraries installed at the paths shown.<br>• The user executing the script has write permission to `${logdir}` and `${reportexportpath}`.<br>• `mailx` is configured for outbound mail. |

---

## 4. Integration Points & Call Flow

| Component | Interaction |
|-----------|-------------|
| **Scheduler (cron / Oozie)** | Invokes `customreportgenerator.sh` with appropriate flags as part of the daily MNAAS reporting pipeline. |
| **`mnaascustomreport.properties`** | Central configuration shared with other MNAAS scripts (e.g., `api_med_data.sh`). |
| **Java Exporter (`tclcustomer.jar`)** | Performs the heavy‑lifting data extraction and CSV generation; other scripts may load the same JAR for different purposes. |
| **Process‑status file** (`MNAAS_Daily_report_traffic_no_dup_ProcessStatusFileName`) | Shared flag file used by multiple scripts to avoid duplicate ticket creation. |
| **Email/SDP system** | On failure, the script raises an incident via `mailx`; downstream ticketing automation consumes these emails. |
| **Potential downstream consumer** | The generated CSVs are later consumed by ingestion scripts (e.g., `api_med_data_oracle_loading.sh`) that load the reports into Oracle or other data warehouses. |
| **Concurrency guard** | The final log line hints that an external lock (perhaps a PID file) prevents overlapping runs; this script respects that guard but does not implement it itself. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Missing or malformed property file | Script aborts early, no logs | Validate existence and required keys at start; exit with clear error if not present. |
| Java classpath or JAR version mismatch | Report generation fails, silent if not logged | Capture Java exit code (already done) and include classpath in log for debugging. |
| Log directory not writable | No audit trail, possible silent failure | Pre‑run check `test -w ${logdir}`; abort with message if false. |
| Large `customreport` / `customerid` lists → resource exhaustion | Long run time, possible OOM | Add throttling (e.g., `sleep` between iterations) or batch processing; monitor runtime. |
| Email/SDP ticket flood on repeated failures | Alert fatigue | Ensure the flag file is correctly updated after first ticket; add rate‑limiting. |
| Concurrency (multiple instances) | Duplicate work, race conditions | Implement a lock file (e.g., `flock`) at script start; exit if lock held. |
| Hard‑coded Hadoop version paths | Breaks after upgrade | Externalize Hadoop base path into a property; use `$(hadoop classpath)` if possible. |

---

## 6. Running / Debugging the Script

### Typical Invocation
```bash
# Bulk generation for all configured reports & customers
./customreportgenerator.sh -r ALL -d 2023-09-30

# Single‑customer, single‑report generation
./customreportgenerator.sh -r RevenueSummary -d 2023-09-30 -s 12345678
```

### Debug Steps
1. **Enable Bash tracing**: prepend `set -x` after the `#!/bin/bash` line (or run `bash -x customreportgenerator.sh …`).  
2. **Check environment**: `echo $logdir $reportexportpath` before execution to confirm property loading.  
3. **Validate Java call**: copy the generated `java` command from the log and run it manually to see stack traces.  
4. **Inspect logs**: `${logdir}/customreportgenerator.sh` contains timestamps, success/failure messages, and the full command line.  
5. **Force failure path**: supply an invalid report name to trigger `terminateCron_Unsuccessful_completion` and verify email/SDP ticket creation.  

---

## 7. External Configuration & Files

| File / Variable | Purpose |
|-----------------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaascustomreport.properties` | Holds all runtime parameters (`logdir`, `reportexportpath`, `customreport`, `customerid`, `setparameter`, status‑file names, `ENV_MODE`, etc.). |
| `setparameter` (from properties) | Expected to be a series of `export VAR=VALUE` statements; `eval $setparameter` materialises them in the script’s environment. |
| `MNAAS_Daily_report_traffic_no_dup_ProcessStatusFileName` | Text file containing flags such as `MNAAS_email_sdp_created=No/Yes`. |
| `MNAAS_Daily_report_traffic_no_dup_scriptName` | Human‑readable script identifier used in log and email messages. |
| `MNAAS_Daily_report_traffic_no_dup_log_file_path` | Full path to the log file; referenced in email body. |
| `tclcustomer.jar` (under `/app/hadoop_users/MNAAS/MNAAS_Jar/Raw_Aggr_tables_processing/`) | Java implementation of `ReportExporterMain`. |
| `mailx` configuration | Required for the `email_and_SDP_ticket_triggering_step` routine. |

---

## 8. Suggested Improvements (TODO)

1. **Add a lock file / `flock` guard** at script start to prevent concurrent executions, replacing the ambiguous “Previous … Process is running already” log message with a deterministic mechanism.  
2. **Externalize Hadoop classpath**: replace the hard‑coded long `-cp` string with `$(hadoop classpath):/app/hadoop_users/MNAAS/MNAAS_Jar/Raw_Aggr_tables_processing/tclcustomer.jar` to simplify upgrades and reduce maintenance overhead.  

---