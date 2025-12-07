**File:** `move-mediation-scripts/bin/MNAAS_Monthly_traffic_nodups_summation.sh`  

---

## 1. High‑Level Summary
This script is the final step of the monthly “traffic‑no‑duplicates” pipeline.  
It (a) runs a Spark job that aggregates the previous month’s de‑duplicated traffic records into a *summation* table, (b) refreshes the corresponding Impala view, and (c) applies a retention policy that drops partitions older than the configured period.  Execution is guarded by a process‑status file to prevent overlapping runs, and any failure triggers an automated email/SDP ticket.

---

## 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `load_data_into_summation_table` | *Sets process flag to “loading”*, computes start/end dates (first‑day of previous month → last‑day of previous month), launches the Spark job (`$MNAAS_Monthly_Traffic_Nodups_Pyfile`) with YARN, and on success runs an Impala `REFRESH` statement. |
| `MNAAS_retention_for_traffic_details_raw_daily_with_no_dups_summation` | *Sets process flag to “retention”*, invokes a Java class (`DropNthOlderPartition`) to drop Hive partitions older than `$mnaas_retention_period_traffic_details_raw_daily_with_no_dups_summation`, then refreshes Impala. |
| `terminateCron_successful_completion` | Writes *Success* status to the process‑status file, logs completion timestamps, and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, calls `email_on_reject_triggering_step`, and exits with code 1. |
| `email_on_reject_triggering_step` | Sends a templated email (via `mailx`) to the Cloudera support mailbox and logs ticket creation. |
| **Main program** | Checks the PID stored in the status file to avoid concurrent runs, reads the flag (`MNAAS_Monthly_ProcessStatusFlag`) and decides which steps to execute (load → retain, or retain only). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Monthly_traffic_nodups_summation.properties` – defines all `$MNAAS_*` variables used throughout the script (paths, hostnames, table names, retention period, Spark/Impala parameters, email addresses, etc.). |
| **External Services** | • **YARN / Spark** – runs the aggregation PySpark job.<br>• **Impala** – `impala-shell` used for `REFRESH` statements.<br>• **Hive / HDFS** – Java class manipulates Hive partitions.<br>• **Mailx / SDP** – sends failure notification email.<br>• **System logger** – `logger -s` writes to syslog and the script‑specific log file. |
| **Files** | • Process‑status file (`$MNAAS_Monthly_Traffic_Nodups_Summation_ProcessStatusFileName`).<br>• Log file (`$MNAAS_Monthly_Traffic_Nodups_Summation_logpath`). |
| **Outputs** | • Populated *summation* table for the previous month (Hive/Impala).<br>• Updated process‑status file (flags, PID, job status).<br>• Log entries and optional email/SDP ticket on failure. |
| **Assumptions** | • The properties file exists and contains all required variables.<br>• Spark, Impala, Hive, Java class, and `mailx` are reachable from the host where the script runs.<br>• The process‑status file is writable and not corrupted.<br>• The previous month’s de‑duplicated traffic data is already present in the source table. |

---

## 4. Interaction with Other Components  

| Component | Relationship |
|-----------|--------------|
| **Up‑stream scripts** (e.g., `MNAAS_Monthly_traffic_nodups_load.sh`, `MNAAS_Monthly_traffic_nodups_transform.sh`) | Produce the raw de‑duplicated traffic tables that this script aggregates. |
| **Down‑stream consumers** (billing, reporting, analytics jobs) | Read the *summation* table refreshed by this script; they expect a single row per MSISDN/period. |
| **Process‑status file** | Shared with other monthly scripts to coordinate execution order and avoid overlap. |
| **Java utility `DropNthOlderPartition`** | Centralised partition‑retention logic used by several monthly pipelines. |
| **SDP ticketing system** | Receives automated incident tickets when the script fails. |
| **Cron scheduler** | Typically invoked nightly/early‑month via a cron entry; the script itself checks for concurrent runs. |

---

## 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – stale PID or missing flag may cause duplicate runs. | Ensure the status file is atomically updated; add a lockfile (`flock`) as an extra safeguard. |
| **Spark job failure** (resource shortage, code bug). | Monitor Spark exit code; configure YARN queues with sufficient resources; add retry logic or alert on non‑zero exit. |
| **Impala refresh failure** (network or metadata lock). | Capture Impala error output; if refresh fails, retry a limited number of times before aborting. |
| **Java partition‑drop error** (wrong retention period, permission). | Validate `$mnaas_retention_period_*` before invoking Java; log the exact command line for post‑mortem. |
| **Missing/incorrect properties** leading to undefined variables. | Add a pre‑flight check that all required `$MNAAS_*` variables are non‑empty; fail fast with a clear message. |
| **Email/SDP delivery failure** masking real failures. | Log the result of `mailx`; optionally fallback to a secondary notification channel (e.g., Slack webhook). |
| **Log file growth** over time. | Rotate logs via `logrotate` or implement size‑based truncation within the script. |

---

## 6. Running / Debugging the Script  

1. **Standard execution (cron)**  
   ```bash
   # Cron entry (example, runs 02:00 on the 2nd day of each month)
   0 2 2 * * /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/MNAAS_Monthly_traffic_nodups_summation.sh >> /dev/null 2>&1
   ```

2. **Manual run (for testing)**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts/bin
   ./MNAAS_Monthly_traffic_nodups_summation.sh
   ```

3. **Debug mode** – the script already enables `set -x`. To capture full trace:  
   ```bash
   bash -x ./MNAAS_Monthly_traffic_nodups_summation.sh 2>&1 | tee /tmp/debug.log
   ```

4. **Inspecting state**  
   - View the process‑status file: `cat $MNAAS_Monthly_Traffic_Nodups_Summation_ProcessStatusFileName`  
   - Check the log: `tail -f $MNAAS_Monthly_Traffic_Nodups_Summation_logpath`  

5. **Force a specific step** (e.g., only retention) by editing the flag in the status file to `2` before running.

---

## 7. External Configuration & Environment Variables  

| Variable (populated from properties) | Purpose |
|--------------------------------------|---------|
| `MNAAS_Monthly_Traffic_Nodups_Summation_ProcessStatusFileName` | Path to the shared status file. |
| `MNAAS_Monthly_Traffic_Nodups_Summation_logpath` | Directory/file for script logs. |
| `MNAAS_Monthly_Traffic_Nodups_Pyfile` | Full path to the Spark Python job. |
| `MNAAS_Monthly_Traffic_Nodups_Summation_Refresh` | Impala `REFRESH` statement executed after each successful step. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Impala/Hive. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `DropNthOlderPartition` | Java classpath and main class for partition retention. |
| `dbname`, `traffic_details_raw_daily_with_no_dups_summation_tblname` | Hive database & table targeted by retention. |
| `mnaas_retention_period_traffic_details_raw_daily_with_no_dups_summation` | Number of months to keep. |
| `SDP_ticket_from_email`, `T0_email` | Email addresses used in failure notifications. |
| `MNAAS_Monthly_Traffic_Nodups_Script` | Human‑readable script identifier used in logs/emails. |

If any of these are missing or empty, the script will likely abort with a cryptic error; a pre‑flight validation step is recommended.

---

## 8. Suggested Improvements (TODO)

1. **Add a pre‑flight validation function** that checks the existence and non‑emptiness of all `$MNAAS_*` variables, and aborts early with a clear log entry if any are missing.  
2. **Replace ad‑hoc PID/status file handling with `flock`** (or a dedicated lock directory) to guarantee atomic single‑instance execution, reducing race‑condition risk when the script is invoked concurrently (e.g., manual run overlapping a scheduled run).  

---