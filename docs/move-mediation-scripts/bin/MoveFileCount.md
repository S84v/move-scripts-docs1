**MoveFileCount.sh – High‑Level Documentation**

---

### 1. Purpose (One‑paragraph Summary)
`MoveFileCount.sh` is a nightly reporting utility that extracts the highest record count per file for the “traffic” processing job from the `mnaas.telena_file_record_count` Impala table for the previous calendar day. It writes the result to a CSV file named `<date>.csv`, emails the file to a predefined distribution list, and then removes the temporary CSV. The script is typically invoked after the daily data‑load pipelines (e.g., the various `MNAAS_*` scripts) have populated the `telena_file_record_count` table.

---

### 2. Key Elements

| Element | Type | Responsibility |
|---------|------|-----------------|
| `date` | Bash variable | Holds the previous day in `YYYY‑MM‑DD` format (used for query filter and CSV filename). |
| `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`, `JDBC_DRIVER_NAME`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `dbname` | Configuration variables | Define the Impala/Hive connection endpoints and target database. |
| `query` | Bash string | Impala SQL that selects `filename` and the maximum `record_count` for each file where `process_name='traffic'` and `file_date` matches `$date`. |
| `impala-shell` command | External tool invocation | Executes the query, outputs a CSV (`$date.csv`) with a header and comma delimiter. |
| `mailx` command | External tool invocation | Sends an email with the CSV attached to a static list of recipients, using a simple body template. |
| `rm $date.csv` | Bash command | Cleans up the temporary CSV after successful mail delivery. |

*No functions or classes are defined; the script is a linear procedural Bash script.*

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - System date (used to compute `$date`).<br>- Impala/Hive cluster reachable at the hard‑coded IPs.<br>- Existing table `mnaas.telena_file_record_count` with columns `filename`, `record_count`, `process_name`, `file_date`.<br>- Local `mailx` configuration (SMTP, authentication). |
| **Outputs** | - CSV file `<date>.csv` containing `filename,records` rows.<br>- Email titled “Move File Count‑Received” with the CSV attached, sent to the static recipient list. |
| **Side‑effects** | - Network traffic to Impala (query execution).<br>- Email generation and transmission.<br>- Deletion of the temporary CSV file. |
| **Assumptions** | - The script runs on a host that can resolve and reach the Impala daemon (`192.168.124.90`).<br>- The Impala service is up and the `mnaas` database is accessible.<br>- The `mailx` command is installed and correctly configured for outbound mail.<br>- The user executing the script has write permission in the current working directory and permission to delete the generated CSV. |
| **External Services** | - **Impala** (SQL query execution).<br>- **Mail Transfer Agent** (via `mailx`). |

---

### 4. Integration Points (How it Connects to Other Scripts/Components)

| Connected Component | Relationship |
|---------------------|--------------|
| **Data‑load scripts** (`MNAAS_*_Load.sh`, `MNAAS_*_tbl_Load.sh`, etc.) | Populate `mnaas.telena_file_record_count`. This script reads the aggregated results after those loads complete. |
| **Scheduler / Cron** | Typically scheduled (e.g., via `cron` or an orchestration tool like Oozie/Airflow) to run after the nightly load window. |
| **Monitoring / Alerting** | May be referenced by operational dashboards that expect the daily “Move File Count” email. |
| **Configuration Management** | The hard‑coded host/IP values are likely mirrored in other `MNAAS_*` scripts; any change to the cluster topology must be applied consistently. |
| **Email Distribution List** | The static list of recipients is shared across other reporting scripts (e.g., `MNAAS_report_data_loading.sh`). |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Impala query failure** (service down, table missing, permission error) | No CSV generated; downstream email empty or not sent. | Capture exit code of `impala-shell`; log error; send an alert email on failure. |
| **Empty result set** (no rows for the previous day) | CSV contains only header; may be misinterpreted as success. | Add a check for file size > header length; if empty, send a “no data” notification. |
| **Mail delivery failure** (SMTP down, auth error) | Stakeholders do not receive the report. | Verify `mailx` return code; retry or fallback to an alternative mail client; log the failure. |
| **Hard‑coded host/IP drift** (cluster changes) | Script points to wrong Impala node, causing timeouts. | Externalize connection parameters to a config file or environment variables; version‑control the config. |
| **File permission issues** (cannot write/delete CSV) | Script aborts, leaving stale files. | Ensure the execution user has appropriate directory permissions; add error handling around `rm`. |
| **Date calculation edge cases** (timezone, DST) | Wrong date used, leading to missing data. | Use UTC consistently or allow an override parameter for the target date. |

---

### 6. Running & Debugging the Script

**Typical Execution**
```bash
cd /path/to/move-mediation-scripts/bin
./MoveFileCount.sh
```

**Debugging Steps**
1. **Enable Verbose Mode** – prepend `set -x` at the top of the script to trace each command.
2. **Check Return Codes** – after `impala-shell` and `mailx`, echo `$?` to verify success.
3. **Inspect Generated CSV** – comment out the `rm $date.csv` line temporarily and view the file.
4. **Log Output** – redirect stdout/stderr to a log file:
   ```bash
   ./MoveFileCount.sh > MoveFileCount_$(date +%F).log 2>&1
   ```
5. **Validate Email** – send a test email to a single address by modifying the `mailx` line.

---

### 7. External Configuration / Environment Variables

| Variable | Current Use | Suggested Externalization |
|----------|-------------|---------------------------|
| `IMPALAD_HOST` | Hard‑coded IP (`192.168.124.90`) | Read from a config file (e.g., `/etc/mnaas/conf.sh`) or environment variable `IMPALAD_HOST`. |
| `IMPALAD_JDBC_PORT`, `JDBC_DRIVER_NAME`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Hard‑coded values | Same as above; keep them in a shared config to avoid duplication across scripts. |
| `dbname` | Fixed to `mnaas` | Parameterizable if multiple databases are used in the future. |
| Email recipients list | Inline in `mailx` command | Move to a config file or environment variable (`MNAAS_REPORT_RECIPIENTS`). |
| `date` calculation | Uses system `date` with `--date='1 day ago'` | Allow an optional argument to override the target date (useful for back‑fills). |

---

### 8. Suggested Improvements (TODO)

1. **Add Robust Error Handling & Logging**  
   - Capture exit status of `impala-shell` and `mailx`.  
   - Write structured logs (timestamp, level, message) to a central log directory.  
   - Send an alert email if any step fails.

2. **Parameterize Configuration**  
   - Externalize all host/port values, database name, and recipient list to a single configuration file or environment variables.  
   - Provide a `--date` command‑line option to specify the reporting date, defaulting to “yesterday”.

---