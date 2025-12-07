**File:** `move-mediation-scripts/bin/MNAAS_report_data_loading.sh`

---

## 1. High‑Level Summary
This Bash driver loads daily “report” CSV files from a staging directory into Hive/Impala tables. For each file it (a) strips the header, (b) truncates a temporary Hive table, (c) loads the raw file into the temp table, (d) truncates the target table, (e) inserts transformed data into the target using a pre‑defined INSERT query, (f) refreshes Impala metadata, (g) validates the load, and (h) archives the processed file. Process status, logging, and failure handling (email + SDP ticket) are driven by a shared status file defined in the accompanying properties file.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_report_data_loading_function`** | Core loop that processes every file found in the source directory: header removal, Hive → temp‑load, target‑load, Impala metadata refresh/validation, file archiving, and per‑step error handling. |
| **`terminateCron_successful_completion`** | Marks the status file as *completed* (`ProcessStatusFlag=0`, `job_status=Success`), logs end‑time, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs abnormal termination, triggers failure‑notification step, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates status file to `job_status=Failure`, checks if an SDP ticket/email has already been raised, and if not sends a templated mail via `mailx` to the support mailbox. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_report_data_loading.properties` – defines:<br>• `$MNAAS_report_data_loading_source_data_location` (staging dir)<br>• `$MNAAS_report_data_loading_backup_data_location` (archive dir)<br>• `$MNAAS_report_data_loading_log_file_path` (log base path)<br>• `$MNAAS_report_data_loading_ProcessStatusFileName` (status file)<br>• `$MNAAS_report_data_loading_scriptName` (script identifier)<br>• `$dataBase_name` (Hive DB)<br>• Associative arrays: `file_format_and_table`, `table_tmpTable_mapping`, `insert_query` |
| **External services** | Hive CLI (`hive -S -e …`), Impala CLI (`impala-shell`), `mailx` (SMTP), syslog (`logger`). |
| **Runtime environment** | `$IMPALAD_HOST` (Impala daemon host), standard Unix utilities (`sed`, `mv`, `ps`, `grep`, `wc`). |
| **Primary input files** | All files present in `$MNAAS_report_data_loading_source_data_location` at script start. |
| **Primary outputs** | • Hive tables populated (temp and target).<br>• Impala metadata refreshed.<br>• Processed files moved to `$MNAAS_report_data_loading_backup_data_location`.<br>• Log entries written to `$MNAAS_report_data_loading_log_file_path$(date +_%F)`.<br>• Status file updated (flags, timestamps, job status). |
| **Side effects** | • Potential SDP ticket creation via email.<br>• System‑wide syslog entries.<br>• Consumption of Hive/Impala resources (temporary table truncation, data load). |
| **Assumptions** | • Property file defines all required variables and associative arrays.<br>• Hive and Impala are reachable and the user has required privileges.<br>• Source files are CSV‑compatible with the expected schema.<br>• Only one instance runs at a time (checked via `ps`). |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Up‑stream** | Files are typically staged by scripts such as `MNAAS_move_files_from_staging_*` which copy raw report files into `$MNAAS_report_data_loading_source_data_location`. |
| **Down‑stream** | Populated Hive tables are consumed by downstream analytics/reporting jobs (e.g., scheduled Spark/Impala queries, BI dashboards). |
| **Shared status file** | `$MNAAS_report_data_loading_ProcessStatusFileName` is read/written by orchestration wrappers (e.g., a master cron controller) to decide whether to launch this script. |
| **Logging** | All scripts in the “move‑mediation” suite write to the same log directory pattern, enabling centralized log aggregation. |
| **Failure escalation** | The `email_and_SDP_ticket_triggering_step` uses a hard‑coded support address; other scripts in the suite use the same mailing pattern for consistency. |

---

## 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Missing or malformed property file** – undefined variables cause script abort. | Validate presence of required keys at start; exit with clear error if any are missing. |
| **Concurrent executions** – race condition on temp tables or status file. | Keep the existing `ps` guard but replace with a lock file (`flock`) for atomicity. |
| **Hive/Impala outage** – truncation or load fails, leaving tables empty. | Add retry logic with exponential back‑off; alert on repeated failures. |
| **Large file causing long load** – may exceed cron window or cause timeouts. | Split large files upstream, or add a configurable timeout/monitor. |
| **Log file growth** – daily log files never rotated. | Implement logrotate or size‑based rotation; archive old logs. |
| **Email flood on repeated failures** – same failure may generate many tickets. | Ensure `MNAAS_email_sdp_created` flag is honoured (already present) and add a cooldown period. |

---

## 6. Running / Debugging the Script

1. **Prerequisite** – Ensure the property file exists and is readable:
   ```bash
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_report_data_loading.properties
   ```
2. **Manual execution** (for testing):
   ```bash
   bash -x move-mediation-scripts/bin/MNAAS_report_data_loading.sh
   ```
   The `-x` flag (already set via `set -x`) prints each command as it runs.

3. **Check status before start**:
   ```bash
   grep ProcessStatusFlag $MNAAS_report_data_loading_ProcessStatusFileName
   ```

4. **Verify source files**:
   ```bash
   ls -l $MNAAS_report_data_loading_source_data_location
   ```

5. **Inspect logs** (today’s log):
   ```bash
   tail -f ${MNAAS_report_data_loading_log_file_path}$(date +_%F)
   ```

6. **Force a single‑run (bypass ps guard)** – useful in dev:
   ```bash
   export MNAAS_report_data_loading_scriptName=dev_test
   bash move-mediation-scripts/bin/MNAAS_report_data_loading.sh
   ```

7. **Debug a specific step** – e.g., Hive load:
   ```bash
   hive -S -e "SHOW TABLES IN ${dataBase_name};"
   ```

8. **Post‑run validation** – query the target table via Impala:
   ```bash
   impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM ${dataBase_name}.${table_name};"
   ```

---

## 7. External Config / Environment Variables

| Variable | Source | Usage |
|----------|--------|-------|
| `MNAAS_report_data_loading_properties` (sourced) | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_report_data_loading.properties` | Provides all runtime paths, DB name, mappings, and flags. |
| `IMPALAD_HOST` | Expected to be exported in the environment or defined in the properties file | Impala daemon host for metadata refresh/validation. |
| `PATH`, `MAILX` | System environment | Required for `hive`, `impala-shell`, `mailx`. |
| `MNAAS_report_data_loading_scriptName` | Property file | Used for logging and status‑file identification. |

If any of these are missing, the script will fail at the first reference (e.g., `sed` on a non‑existent file).

---

## 8. Suggested Improvements (TODO)

1. **Add explicit validation of the property file** – iterate over a required‑key list and abort with a clear message if any are undefined.
2. **Replace the `ps` concurrency guard with a lock file (`flock`)** – guarantees atomic single‑instance execution even across rapid restarts or overlapping cron schedules.