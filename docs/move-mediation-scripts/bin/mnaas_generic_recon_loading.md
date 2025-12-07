**File:** `move-mediation-scripts\bin\mnaas_generic_recon_loading.sh`  

---

## 1. Purpose (one‑paragraph summary)

`mnaas_generic_recon_loading.sh` orchestrates the end‑to‑end loading of *generic reconciliation* files into the MNAAS data‑warehouse. It moves inbound files from a landing directory to daily or monthly intermediate folders, copies them to HDFS, loads them into a temporary Hive table via a Java MapReduce/JAR job, refreshes the table in Impala, drops and re‑creates partitioned Hive tables, backs up the processed files, generates a CSV reconciliation report, and finally sends a success/failure email (and creates an SDP ticket) based on the run outcome. The script maintains its own state machine in a shared “process‑status” file so that it can resume from the appropriate step if re‑executed.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `start()` | Sources the property file `mnaas_generic_recon_loading.properties`, redirects STDERR to the script log, and stores the four command‑line arguments (`processname`, `filelike`, `filetype`, `emailtype`). |
| `mnaas_generic_recon_enrich()` | Updates the status flag to **1**, moves matching files from the landing dir (`$mnaasdatalandingdir/$filelike`) to the appropriate intermediate folder (`daily` or `monthly`). |
| `mnaas_generic_recon_temp_table_load()` | Updates status flag to **2**, copies intermediate files to HDFS (`$mnaas_recon_load_generic_pathname`), runs the Java class `$Load_nonpart_table` to populate a temporary Hive table, and refreshes it in Impala. |
| `mnaas_generic_recon_loading()` | Updates status flag to **3**, drops existing Hive partitions (using queries from the properties file), inserts new partitions from the temp table, and refreshes the final Impala view/table. |
| `mnaas_copy_recon_files_to_backup()` | Updates status flag to **4**, moves processed intermediate files to a backup directory (`$generic_backupdir`). |
| `mnaas_recon_report_files()` | Updates status flag to **5**, runs a per‑partition Impala query (built from property templates) to generate a CSV report (`$mnaas_generic_recon_report_files`). |
| `terminateCron_successful_completion()` | Resets status flag to **0**, writes success metadata (run time, job status) to the status file, optionally sends a success email, logs termination, and exits 0. |
| `terminateCron_Unsuccessful_completion()` | Logs failure, writes failure metadata, triggers email + SDP ticket, and exits 1. |
| `email_success_triggering_step()` | Sends a success email with the CSV report attached (via `mailx`). |
| `email_and_SDP_ticket_triggering_step()` | Sends a failure email and creates an SDP ticket (via `mailx`). |
| **Main driver** (while‑getopts + flag logic) | Parses CLI options, validates required arguments, prevents concurrent runs using the PID stored in the status file, and drives the state‑machine based on the current flag value. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **CLI arguments** | `-p <processname>` – logical name of the run (used only for logging.<br>`-f <filelike>` – shell glob (e.g. `*.txt`) that selects files in the landing directory.<br>`-t <filetype>` – either `daily` or `monthly` (controls target folders).<br>`-e <emailtype>` – `yes` to trigger email notifications. |
| **Configuration file** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaas_generic_recon_loading.properties` – defines all paths, DB names, table names, Hive/Impala queries, email recipients, etc. |
| **External services** | • Hadoop HDFS (`hdfs dfs`, `hadoop fs`)<br>• Hive (`hive -e`)<br>• Impala (`impala-shell`)<br>• Java runtime (JAR `$Load_nonpart_table`)<br>• Syslog (`logger`)<br>• Mail system (`mailx`) – used for both success and failure notifications.<br>• SDP ticketing (via email to `Cloudera.Support@tatacommunications.com`). |
| **Files created / modified** | • Script log: `$mnaas_generic_recon_loadinglogname` (stderr + explicit logger calls).<br>• Process‑status file: `$mnaas_generic_recon_processstatusfilename` (holds flags, PID, timestamps, email‑sent flag).<br>• Intermediate folders (`$mnaasinterfilepath_*_generic_recon_details`).<br>• HDFS load directory (`$mnaas_recon_load_generic_pathname`).<br>• Backup directory (`$generic_backupdir`).<br>• CSV report file (`$mnaas_generic_recon_report_files`). |
| **Side effects** | • Moves/renames physical files on the local filesystem.<br>• Writes to HDFS.<br>• Alters Hive/Impala tables (drop/insert partitions).<br>• Sends email / creates SDP tickets.<br>• Updates shared status file (potential concurrency point). |
| **Assumptions** | • All required environment variables are defined in the sourced properties file.<br>• Hadoop, Hive, Impala, Java, and `mailx` are installed and reachable from the host.<br>• The user running the script has write permission on all local and HDFS paths.<br>• The status file is the single source of truth for process coordination. |

---

## 4. Interaction with Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **`mnaas_checksum_validation.sh`** | Likely runs **before** this script to verify inbound file integrity; this script expects clean files in `$mnaasdatalandingdir`. |
| **`mnaas_generic_recon_loading.properties`** | Central configuration shared with other MNAAS recon scripts (e.g., IPV‑Probe recon). |
| **Scheduler / Cron** | The script is invoked by a cron job (or an orchestration tool) with the required flags; the PID‑check prevents overlapping runs. |
| **Java loader (`Load_nonpart_table`)** | External JAR invoked to bulk‑load the temporary Hive table; other recon scripts use the same loader with different parameters. |
| **Email / SDP ticket system** | Uses the same mailing conventions as other MNAAS scripts (`email_success_triggering_step`, `email_and_SDP_ticket_triggering_step`). |
| **Backup / Archive processes** | The backup directory may be consumed by downstream archival jobs or manual audit processes. |
| **Reporting dashboards** | The generated CSV (`$mnaas_generic_recon_report_files`) is likely consumed by BI tools or operational dashboards. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – PID check uses a simple grep; a stale PID could block future runs. | Implement a robust lock file (e.g., `flock`) and verify that the PID is still alive before aborting. |
| **Status‑file race conditions** – Multiple scripts edit the same file with `sed`. | Use atomic writes (`mv` a temp file) or a dedicated key‑value store (e.g., Zookeeper) for state. |
| **Partial file movement** – If the script crashes after moving files but before loading, data may be lost. | Add a “staging” sub‑directory and only move to final location after successful HDFS copy; keep a checksum log. |
| **Hard‑coded paths / missing properties** – Failure if any property is undefined. | Validate all required variables after sourcing the properties file; fail fast with clear messages. |
| **Silent failures of external commands** – Some commands only check `$?` after a block; others are ignored. | Wrap each external call in a helper function that logs and aborts on non‑zero exit codes. |
| **Email/SDP ticket spam** – Re‑tries could generate duplicate tickets. | Record a “ticket‑sent” flag in the status file and guard against re‑sending within the same run. |
| **Resource exhaustion** – Large file sets could overwhelm local disk or HDFS. | Add size/count thresholds and alert if limits are exceeded before proceeding. |

---

## 6. Typical Execution & Debugging Steps

1. **Prepare environment** – Ensure the properties file exists and contains correct values (paths, DB names, email addresses).  
2. **Run manually (dry‑run)**:  

   ```bash
   ./mnaas_generic_recon_loading.sh -p recon_generic -f "*.txt" -t daily -e yes
   ```

   *The script prints usage errors if any argument is missing.*

3. **Monitor progress** – Tail the log file defined by `$mnaas_generic_recon_loadinglogname`:

   ```bash
   tail -f /path/to/mnaas_generic_recon_loading.log
   ```

4. **Check status flag** – View the process‑status file to see the current step:

   ```bash
   grep MNAAS_Daily_ProcessStatusFlag $mnaas_generic_recon_processstatusfilename
   ```

5. **Debug failures** – Because `set -x` is enabled, the shell prints each command. If a step fails, the log will contain the failing command and its exit code. Re‑run the failing function individually (e.g., `mnaas_generic_recon_temp_table_load`) after sourcing the properties file.

6. **Verify Hive/Impala tables** – After a successful run, run:

   ```bash
   impala-shell -i $IMPALAD_HOST -q "SHOW TABLES IN mnaas LIKE '${generic_recon_inter_tblname}*';"
   ```

7. **Confirm email/SDP ticket** – Check the mailbox of `Cloudera.Support@tatacommunications.com` and the log for “SDP ticket created”.

---

## 7. External Configurations & Environment Variables

| Variable (populated from properties) | Meaning |
|--------------------------------------|---------|
| `mnaas_generic_recon_loadinglogname` | Path to the script’s log file. |
| `mnaas_generic_recon_processstatusfilename` | Shared status/flag file. |
| `mnaasdatalandingdir` | Directory where inbound files land. |
| `filelike` (CLI) | Glob pattern for files to process. |
| `mnaasinterfilepath_daily_generic_recon_details` / `...monthly...` | Intermediate folders for daily/monthly files. |
| `mnaas_recon_load_generic_pathname` | HDFS target directory for the load. |
| `Load_nonpart_table` | Java class name invoked via the JAR. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath & JAR list. |
| `dbname`, `generic_recon_inter_tblname`, `Generic_daily_recon_tblname` | Hive database and table names. |
| `generic_partition_date_query`, `generic_drop_partitions`, `generic_insert_partitions` | Hive SQL snippets used for partition management. |
| `generic_backupdir` | Local backup directory for processed files. |
| `mnaas_generic_recon_report_files` | Full path of the CSV report generated. |
| `mnaas_generic_header` | Header line written to the CSV. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Impala/Hive. |
| `SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM`, `MOVE_TEAM_EMAIL` | Email routing for notifications/tickets. |
| `emailtype` (CLI) | Controls whether success email is sent. |

If any of these variables are missing or empty, the script will likely abort with a “command not found” or “file not found” error.

---

## 8. Suggested Improvements (TODO)

1. **Replace ad‑hoc `sed` status updates with atomic writes** – create a temporary file, write the new key/value pairs, then `mv` it over the original. This eliminates race conditions when multiple scripts touch the same file.

2. **Introduce a proper lock mechanism** – use `flock` or a lock‑file in `/var/lock/mnaas_generic_recon_loading.lock` to guarantee single‑instance execution and to clean up stale locks automatically.

---