**High‑Level Documentation – `move-mediation-scripts/bin/processAsyncBackup.sh`**

---

### 1. Purpose (one‑paragraph summary)

`processAsyncBackup.sh` is a thin wrapper that triggers the **AsyncNotification** Java component used by the MNAAS (Move‑Network‑As‑a‑Service) mediation platform. It loads a shared property file, then executes `AsyncNotification.jar` with three arguments: the mode flag (`File`), the same property file path, and a dedicated log file. In production the script is scheduled (typically via cron) to process pending asynchronous notifications that have been persisted to a staging area, generate any required downstream messages, and record its activity in `MoveAsyncBNotification.log`.

---

### 2. Key Artifacts

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `processAsyncBackup.sh` | Bash wrapper | Sources environment/property definitions and launches the Java notification processor. |
| `MNAAS_ShellScript.properties` | Property file (key‑value) | Supplies configuration for the Java jar – e.g., DB/JDBC URLs, Hadoop/HDFS paths, SFTP/FTP endpoints, API tokens, thread pool sizes, and log level. |
| `AsyncNotification.jar` | Java executable (contains `AsyncNotification` main class) | Implements the business logic for reading staged async notification records, transforming them, and dispatching to target systems (queues, APIs, files). |
| `MoveAsyncBNotification.log` | Log file (plain text) | Captures stdout/stderr from the Java process; used for operational monitoring and troubleshooting. |

*No functions or classes are defined inside the shell script itself; the heavy lifting resides in the Java `AsyncNotification` class.*

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_ShellScript.properties` (absolute path) – must be readable.<br>• Implicit input data sources referenced inside the properties (e.g., HDFS staging directory, relational DB tables, message queues). |
| **Outputs** | • `MoveAsyncBNotification.log` – appended with execution details, errors, and summary statistics.<br>• Side‑effects performed by the Java jar: creation/consumption of files in HDFS, updates to DB tables (e.g., marking notifications as processed), pushes to external APIs or messaging systems (Kafka, JMS, etc.). |
| **Assumptions** | • Java runtime (compatible JRE version) is installed and on `$PATH`.<br>• The property file contains all required keys; missing keys will cause the jar to abort.<br>• The script runs under a user with sufficient permissions to read the property file, write the log, and access any external resources (HDFS, DB, network). |
| **External Services** | • Hadoop/HDFS cluster (for staging files).<br>• Relational database(s) (e.g., Oracle, PostgreSQL) for notification state.<br>• Message brokers or REST endpoints for downstream delivery.<br>• Possibly SFTP/FTPS servers if the async payload is written to files. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | How `processAsyncBackup.sh` ties in |
|---------------------|--------------------------------------|
| **`mnaas_move_files_from_staging_genric.sh`** | That script moves raw files from a staging area into HDFS. `processAsyncBackup.sh` later consumes the same staged data (via the Java jar) to generate async notifications. |
| **`mnaas_tbl_load_generic.sh` / `mnaas_parquet_data_load.sh`** | These scripts load transformed data into Hive/Impala tables. The async notification processor may read from those tables to decide which notifications to emit. |
| **`move_table_compute_stats.sh`** | Computes statistics that could be included in notification payloads; runs before or after the async job depending on SLA. |
| **Cron Scheduler** | Typically invoked by a cron entry (e.g., `0 * * * * /path/processAsyncBackup.sh`) to ensure periodic processing. |
| **Logging/Monitoring** | Log file is consumed by log aggregation tools (Splunk, ELK) and may trigger alerts on error patterns. |
| **`AsyncNotification.jar`** | Central Java component shared across multiple wrapper scripts (e.g., a “real‑time” version may be called by another script). |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or malformed property file** | Jar aborts; no notifications sent. | Validate existence and syntax of `MNAAS_ShellScript.properties` before launching; exit with clear error code. |
| **Java process crashes / OOM** | Partial processing, possible data loss. | Run the jar with JVM memory limits (`-Xmx`) defined in the properties; monitor exit code; implement retry/back‑off logic in the wrapper if needed. |
| **Log file growth** | Disk exhaustion on the node. | Rotate `MoveAsyncBNotification.log` via logrotate or a built‑in size‑based rollover in the Java code. |
| **Permission issues** (read/write/HDFS/DB) | Silent failures or permission denied errors. | Ensure the executing user belongs to required OS groups and has DB/HDFS ACLs; test with a dry‑run mode if available. |
| **External service latency or outage** (e.g., API endpoint down) | Processing stalls, timeouts. | Configure reasonable timeouts in the jar; implement circuit‑breaker pattern; alert on repeated timeouts. |
| **Version drift** (property file updated but jar not rebuilt) | Incompatible config leads to runtime errors. | Enforce version coupling via CI/CD pipeline; tag both property file and jar with matching version identifiers. |

---

### 6. Running / Debugging the Script

**Typical execution (operator):**
```bash
# As the MNAAS service user
cd /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars
./processAsyncBackup.sh
```
- The script writes progress to `/app/hadoop_users/MNAAS/MNAASCronLogs/MoveAsyncBNotification.log`.
- Verify success by checking the exit status (`echo $?`) and scanning the log for “Completed” or “ERROR”.

**Debugging steps:**
1. **Enable Bash tracing** – prepend `set -x` before the `java -jar` line or run `bash -x processAsyncBackup.sh`.
2. **Check property file** – `cat /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` and verify required keys (e.g., `async.input.path`, `db.url`).
3. **Run the jar manually** to capture stdout/stderr directly:
   ```bash
   java -jar AsyncNotification.jar File \
        /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties \
        /tmp/debug.log
   tail -f /tmp/debug.log
   ```
4. **Inspect Java exit code** – after execution, `echo $?` should be `0`. Non‑zero indicates failure; consult the log for stack traces.
5. **Check external dependencies** – use `hdfs dfs -ls <input_path>` or `sqlplus`/`psql` to confirm DB connectivity.

---

### 7. External Configuration & Environment Variables

| Item | Usage |
|------|-------|
| `MNAAS_ShellScript.properties` (sourced) | Provides variables such as `JAVA_HOME`, `HADOOP_CONF_DIR`, DB credentials, HDFS paths, API endpoints, and any mode flags required by `AsyncNotification.jar`. |
| Implicit environment (e.g., `PATH`, `LD_LIBRARY_PATH`) | Must contain the Java binary compatible with the jar. |
| Cron‑specific variables (e.g., `MAILTO`) | Not used directly but may affect notification on failure. |

*If the property file references other files (e.g., keystore, credential files), those must be present and readable.*

---

### 8. Suggested Improvements (TODO)

1. **Add pre‑flight validation** – before invoking the jar, script should verify that the property file exists, is readable, and contains a minimal set of required keys (e.g., `async.input.path`). Exit with a distinct error code if validation fails.
2. **Implement log rotation** – either integrate a `logrotate` config for `MoveAsyncBNotification.log` or modify the Java component to rotate logs based on size/time, preventing uncontrolled disk growth.

---