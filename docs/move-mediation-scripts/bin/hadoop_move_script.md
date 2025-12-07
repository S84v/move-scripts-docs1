**File:** `move-mediation-scripts\bin\hadoop_move_script.sh`  

---

## 1. High‑Level Summary
This Bash utility performs a simple HDFS “rotation” for PGW (Packet Gateway) mediation output. It checks the number of files in the HDFS directory `/user/DRB/PGW/ValidatedOutput`.  
* If the directory is empty, it restores the most recent eight files from the backup directory `/user/DRB/PGW/ValidatedOutput_backup` back into the live folder.  
* If the directory already contains data, it logs the count; when the count exceeds nine, it moves the five newest files from the live folder into the backup folder.  

All actions are logged to a per‑day log file under `/app/hadoop_users/mkoripal/PGWHadoopmoveLog`. The script is invoked directly (no arguments) and runs once per execution.

---

## 2. Key Functions & Responsibilities
| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `hadoop_move()` | Bash function | Implements the file‑count check, conditional move‑back from backup, conditional archiving to backup, and logging. |
| `set -x` | Bash option | Enables command‑trace debugging for every run (useful for operators). |
| `logger -s` | System logger | Writes timestamped messages to both syslog and the script‑specific log file. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - HDFS directories: `/user/DRB/PGW/ValidatedOutput` and `/user/DRB/PGW/ValidatedOutput_backup`.<br>- Local log directory: `/app/hadoop_users/mkoripal/PGWHadoopmoveLog` (must be writable). |
| **Outputs** | - No stdout data (except the `echo` of each moved file path).<br>- Log entries appended to `$PGWHadoopmoveLog$(date +_%F)`. |
| **Side Effects** | - Moves HDFS files between the two directories (`hdfs dfs -mv`).<br>- Potentially changes downstream processing order (files removed from live folder). |
| **Assumptions** | - Hadoop client (`hdfs`) is installed and configured for the user running the script.<br>- The script runs with sufficient HDFS permissions to list and move files in the two directories.<br>- The backup directory always contains at least eight files when a restore is needed.<br>- The environment variable `PGWHadoopmoveLog` is defined inside the script (hard‑coded path). |
| **External Services** | - HDFS NameNode / DataNode cluster.<br>- Syslog daemon (via `logger`). |
| **Dependencies** | - No external configuration files; all paths are hard‑coded. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | Interaction Pattern |
|---------------------|---------------------|
| **Up‑stream mediation jobs** (e.g., `cdr_buid_search_loading.sh`, `compression.sh`, `customer_invoice_summary_estimated.sh`) | Produce validated PGW output files in `/user/DRB/PGW/ValidatedOutput`. This script may later archive those files when the count grows. |
| **Down‑stream consumers** (e.g., reporting, billing, or ETL jobs) | Expect a bounded number of files in the live folder; the rotation performed here prevents uncontrolled growth that could impact downstream processing. |
| **Scheduler / Orchestrator** (e.g., cron, Airflow, or a custom job‑balancer) | Likely invoked on a regular cadence (hourly/daily) to keep the folder size in check. The script’s `set -x` output can be captured by the scheduler’s logs for troubleshooting. |
| **Monitoring / Alerting** | Log file `$PGWHadoopmoveLog_YYYY-MM-DD` can be tailed by monitoring tools to detect abnormal counts or move failures. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded paths** – changes to HDFS layout or log location require script edit. | Deployment friction, possible silent failures. | Externalize paths via environment variables or a small config file; document required variables. |
| **Numeric comparison using `==`** – Bash treats `==` as string comparison; may mis‑behave on some shells. | Incorrect branch execution, e.g., treating “0” as non‑zero. | Replace with `-eq` for equality and `-gt` for greater‑than. |
| **No error handling** – `hdfs dfs -mv` failures are not captured; script continues. | Lost files, inconsistent state between live and backup. | Capture exit codes, abort on error, and log failure details. |
| **Race condition** – simultaneous runs could move the same files twice. | Duplicate moves, possible data loss. | Use a lock file (`flock`) or ensure the orchestrator runs only one instance at a time. |
| **Assumes at least 8 files in backup** – if backup is empty, the `for` loop will attempt to move nothing, but the script may still log “restored”. | Misleading logs, downstream jobs may wait for data that never arrives. | Verify backup count before restore; log a warning if insufficient. |
| **`set -x` always on** – verbose output may fill logs quickly. | Log bloat, harder to spot real errors. | Make tracing optional via a `DEBUG` env variable. |

---

## 6. Running / Debugging the Script  

1. **Standard execution** (as the Hadoop user that owns the directories):  
   ```bash
   /app/hadoop_users/mkoripal/move-mediation-scripts/bin/hadoop_move_script.sh
   ```
2. **Check the log** for the current day:  
   ```bash
   tail -f /app/hadoop_users/mkoripal/PGWHadoopmoveLog_$(date +%F)
   ```
3. **Debug mode** – temporarily disable `set -x` or wrap the call with `bash -x` to see expanded trace:  
   ```bash
   bash -x hadoop_move_script.sh
   ```
4. **Validate HDFS state before/after**:  
   ```bash
   hdfs dfs -ls /user/DRB/PGW/ValidatedOutput
   hdfs dfs -ls /user/DRB/PGW/ValidatedOutput_backup
   ```
5. **Force a dry‑run** (no moves) by commenting out the `hdfs dfs -mv` lines and observing the log output.

---

## 7. External Config / Environment Variables  

| Variable | Current Definition | Intended Use |
|----------|-------------------|--------------|
| `PGWHadoopmoveLog` | Hard‑coded in script: `/app/hadoop_users/mkoripal/PGWHadoopmoveLog` | Base path for daily log files. |
| `HADOOP_CONF_DIR`, `HADOOP_USER_NAME`, etc. | Not referenced directly but required by the Hadoop client. | Provide Hadoop client configuration; assumed to be set in the user’s environment. |

If the deployment moves to a different environment, these values should be externalized (e.g., via a `.env` file sourced at script start).

---

## 8. Suggested Improvements (TODO)

1. **Refactor for robustness** – replace back‑ticks with `$(…)`, add proper numeric comparisons (`-eq`, `-gt`), and implement error handling for each `hdfs dfs -mv` operation.
2. **Parameterize paths** – read source, backup, and log directories from environment variables or a small JSON/YAML config file, enabling reuse across environments without code changes.  

---