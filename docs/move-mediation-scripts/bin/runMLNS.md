**High‑Level Documentation – `move-mediation-scripts/bin/runMLNS.sh`**

---

### 1. Summary
`runMLNS.sh` is a thin orchestration wrapper that triggers the **MLNS** data‑load Java process. It sources the central MNAAS property file to obtain configuration values, then executes `MLNS.jar` with two arguments: the same property file and a dedicated log file (`MoveMLNS.log`). In production the script is scheduled (typically via cron) to ingest, transform, and persist MLNS‑related records into the mediation data warehouse.

---

### 2. Key Components & Responsibilities
| Component | Type | Responsibility |
|-----------|------|----------------|
| **`/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`** | Config file (sourced) | Provides environment variables, DB connection strings, file locations, and other runtime parameters used by the Java loader. |
| **`MLNS.jar`** | Java executable (loader) | Implements the actual data‑movement logic for the MLNS domain: reads source data (e.g., HDFS, DB, SFTP), applies required transformations, and writes to target tables. |
| **`/app/hadoop_users/MNAAS/MNAASCronLogs/MoveMLNS.log`** | Log file | Captures stdout/stderr of the Java process for audit and troubleshooting. |
| **`runMLNS.sh`** | Bash wrapper | Sources the property file, invokes the Java loader, and provides a single entry point for scheduling/monitoring. |

*Note:* A commented line shows a previous/alternative loader (`BARepLoaderJarPath`). This indicates that the script follows a common pattern used across the suite (e.g., `runGBS.sh`, `runBARep.sh`).

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_ShellScript.properties` (environment variables, DB URLs, credentials, file paths).<br>• Implicit inputs consumed by `MLNS.jar` (source files, tables, or streams) as defined in the property file. |
| **Outputs** | • `MoveMLNS.log` – execution log.<br>• Data persisted by `MLNS.jar` into target mediation tables (exact tables defined in the Java code/config). |
| **Side‑Effects** | • Network I/O to source systems (e.g., SFTP, DB, HDFS).<br>• Writes to target database(s) and possibly to message queues if the loader publishes events. |
| **Assumptions** | • Java 1.8+ is installed and `java` is on the PATH.<br>• The property file exists and is readable by the script user.<br>• `MLNS.jar` is present at the hard‑coded location and executable.<br>• Required external services (DB, HDFS, SFTP) are reachable and credentials are valid.<br>• Sufficient disk space for temporary files/logs. |

---

### 4. Integration Points & Call Graph

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| **`runGBS.sh`, `runBARep.sh`, `runGBS.sh`** | Same orchestration pattern; likely invoked sequentially or in parallel by a master cron or workflow engine. |
| **Cron Scheduler** | `runMLNS.sh` is typically scheduled (e.g., nightly) via a crontab entry under the MNAAS user. |
| **MNAAS Property Loader** (`MNAAS_ShellScript.properties`) | Shared across all loader scripts; central source of configuration. |
| **Data Sources** (e.g., HDFS directories, SFTP drop zones) | Defined in the property file; `MLNS.jar` reads from them. |
| **Target Database** (e.g., Oracle, Hive, PostgreSQL) | Populated by `MLNS.jar`; other scripts may read from the same tables for downstream reporting. |
| **Monitoring / Alerting** (e.g., Nagios, Splunk) | Consumes `MoveMLNS.log` for health checks; not directly referenced but typical in the environment. |

*If a workflow orchestrator (e.g., Oozie, Airflow) is used, `runMLNS.sh` would be wrapped as a shell action node.*

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or corrupted `MLNS.jar`** | Loader fails, no data loaded. | Add a pre‑run existence check (`[ -f "$JAR" ] || { echo "Jar missing"; exit 1; }`). |
| **Invalid or stale property values** (e.g., wrong DB password) | Authentication errors, data loss. | Validate critical properties after sourcing; rotate credentials via a secure vault. |
| **Java OOM / long‑running process** | Job hangs, consumes all memory. | Tune JVM options (e.g., `-Xmx`) inside the jar or wrapper; add timeout logic (`timeout` command). |
| **Log file growth** | Disk exhaustion. | Rotate logs (logrotate) and enforce size limits. |
| **Network/service outage** (DB, HDFS, SFTP) | Partial load, retries needed. | Implement retry logic in the Java loader; monitor upstream services. |
| **Permission issues** (script or jar not executable) | Immediate failure. | Ensure correct Unix permissions (`chmod +x` for script, readable for jar). |
| **Uncaptured exit status** | Scheduler assumes success while job failed. | Propagate Java exit code (`java -jar … || exit $?`). |

---

### 6. Running & Debugging Guide

1. **Manual Execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars
   /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/runMLNS.sh
   ```
   - Verify that `MoveMLNS.log` is created/updated in `/app/hadoop_users/MNAAS/MNAASCronLogs/`.

2. **Check Exit Code**  
   ```bash
   echo $?   # 0 = success, non‑zero = failure
   ```

3. **Inspect Logs**  
   ```bash
   tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/MoveMLNS.log
   ```

4. **Validate Configuration**  
   ```bash
   grep -E 'DB|HDFS|SFTP' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   ```

5. **Debug Java Loader**  
   - If deeper inspection is needed, run the jar directly with `-Dlog4j.debug=true` (if supported) or attach a remote debugger using JVM options (`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005`).

6. **Cron Verification**  
   - Check the crontab entry: `crontab -l | grep runMLNS.sh`.  
   - Ensure the environment for cron matches interactive runs (source `/etc/profile` if needed).

---

### 7. External Config / Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| **`MNAAS_ShellScript.properties`** (sourced) | Defines all runtime parameters: DB URLs, credentials, source file locations, Hadoop/HDFS paths, logging directories, etc. |
| **`JAVA_HOME`** (implicit) | Determines which Java runtime is used. |
| **`PATH`** (implicit) | Must contain the `java` executable. |
| **`MLNS.jar`** (hard‑coded path) | The actual loader implementation; any change requires script update. |

*If additional env vars are referenced inside the property file (e.g., `MNAAS_DB_USER`), they are inherited from the shell environment.*

---

### 8. Suggested Improvements (TODO)

1. **Add Robust Error Handling**  
   - Verify the existence and readability of the property file and jar before execution.  
   - Capture the Java process exit code and log a concise success/failure message; exit the script with the same code for proper scheduler handling.

2. **Parameterize Paths**  
   - Replace hard‑coded absolute paths with variables (e.g., `MLNS_JAR=${MLNS_JAR:-/app/.../MLNS.jar}`) and allow overrides via environment or command‑line arguments. This eases migration between environments (dev/test/prod).

---