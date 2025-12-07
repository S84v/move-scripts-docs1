**File:** `move-mediation-scripts/bin/startMoveConsumer.sh`

---

## 1. High‑Level Summary
`startMoveConsumer.sh` is a thin wrapper that launches the **Move Notification Consumer** Java process (`Notification.jar`). It loads the shared MNAAS property file, then starts the consumer in “Consumer” mode, directing its runtime log to the central Move‑Notification log file. In production this script is used (typically via cron or an orchestration tool) to keep the Move‑Notification consumer continuously running, pulling messages from the internal messaging layer (e.g., Kafka/JMS) and triggering downstream data‑move workflows.

---

## 2. Key Components & Responsibilities  

| Component | Type | Responsibility |
|-----------|------|----------------|
| `MNAAS_ShellScript.properties` | Configuration file (sourced) | Provides environment variables, paths, and runtime parameters required by the Java consumer (e.g., broker URLs, DB credentials, Hadoop settings). |
| `Notification.jar` | Java executable (fat‑jar) | Contains the `Notification` class with a `main` method that implements three modes; this script invokes the **Consumer** mode which subscribes to the Move‑Notification topic/queue and processes each message. |
| `MoveNotification.log` | Log file | Captures stdout/stderr of the Java process; used for operational monitoring and troubleshooting. |
| `startMoveConsumer.sh` | Bash wrapper | Sources the property file, validates required files, and launches the Java consumer with the correct arguments. |

*Note:* The Java class `Notification` is also used by other scripts (e.g., `startAsyncMoveConsumer.sh`, `startBalMoveConsumer.sh`) in different modes; this script always selects the **Consumer** mode.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | 1. `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` (must exist & be readable).<br>2. `Notification.jar` located at `/app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/Notification.jar`.<br>3. Runtime environment variables (e.g., `JAVA_HOME`, `PATH`). |
| **Outputs** | 1. Log file `/app/hadoop_users/MNAAS/MNAASCronLogs/MoveNotification.log` (appended).<br>2. Side‑effect: Consumer process creates/updates records in downstream systems (DB tables, HDFS, S3) as defined inside `Notification.jar`. |
| **Side‑Effects** | - Network I/O to the messaging broker (Kafka/JMS).<br>- Potential writes to Hadoop/HDFS, relational DBs, or other downstream services (defined in the property file). |
| **Assumptions** | - The property file contains all required keys (broker URL, consumer group, DB connection strings).<br>- The Java runtime is compatible with the jar (Java 8+).<br>- The executing user (`MNAAS`) has read permission on the property file and execute permission on the jar, and write permission on the log directory. |

---

## 4. Integration with Other Scripts & Components  

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| `startAsyncMoveConsumer.sh` / `startBalMoveConsumer.sh` | Parallel consumers that run the same `Notification.jar` in different modes (e.g., async processing, balance updates). All share the same property file, so changes affect every consumer. |
| `MNAAS_move_files_from_staging.sh`, `MNAAS_Weekly_KYC_Feed_Loading.sh`, etc. | Downstream batch jobs that are **triggered** by messages processed by this consumer (e.g., a “move‑notification” message may cause the KYC feed loader to run). |
| Cron / Scheduler (e.g., `/etc/cron.d/mnaas`) | Typically invokes `startMoveConsumer.sh` at system boot or on a restart schedule to ensure the consumer is always alive. |
| Monitoring / Alerting (e.g., Splunk, ELK) | Consumes `MoveNotification.log` for health checks and error detection. |
| External services (Kafka/JMS broker, Hadoop/HDFS, RDBMS) | Configured via the sourced property file; the consumer reads from the broker and writes to these stores. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or corrupted property file** | Consumer fails to start or runs with wrong parameters. | Add a pre‑flight check (`[ -f $PROP_FILE ] || exit 1`) and alert on failure. |
| **Jar not found / version mismatch** | Process aborts; downstream pipelines stall. | Verify checksum at deployment; use a symbolic link to a version‑controlled directory. |
| **Log file growth** | Disk exhaustion → consumer crash. | Rotate logs via `logrotate` (daily, keep 7 days) and monitor disk usage. |
| **Consumer process dies silently** | No move notifications processed. | Run under a process supervisor (systemd, supervisord) with auto‑restart and health‑check scripts. |
| **Incorrect broker credentials** | No messages consumed, possible security lockout. | Store credentials in a secure vault; reload properties on credential rotation. |
| **Permission issues** (read/write) | Startup failure. | Ensure the `MNAAS` user owns the script, property file, jar, and log directory. |

---

## 6. Running & Debugging Guide  

1. **Standard start (operator)**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_ShellScript
   ./move-mediation-scripts/bin/startMoveConsumer.sh
   ```
   The script returns immediately; the Java process runs in the foreground (stdout/stderr redirected to the log).

2. **Check that the process is alive**  
   ```bash
   ps -ef | grep Notification.jar | grep Consumer
   ```

3. **Tail the log for real‑time output**  
   ```bash
   tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/MoveNotification.log
   ```

4. **Debugging steps**  
   - Verify the property file exists and is readable.  
   - Run the Java command manually to see any stack traces:  
     ```bash
     java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/Notification.jar Consumer \
          /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties \
          /tmp/debug.log
     ```
   - Check Java version: `java -version`.  
   - Inspect broker connectivity (e.g., `nc -z <broker-host> <port>`).  

5. **Stopping the consumer**  
   ```bash
   pkill -f "Notification.jar.*Consumer"
   ```

---

## 7. External Configuration & Environment Variables  

| Variable / File | Purpose | Typical Location |
|-----------------|---------|------------------|
| `MNAAS_ShellScript.properties` | Central configuration (broker URLs, DB creds, Hadoop settings). | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` |
| `JAVA_HOME` (env) | Determines which Java runtime is used. | Exported in the user’s profile or the property file. |
| `PATH` (env) | Must include `$JAVA_HOME/bin`. | System profile. |
| `HADOOP_CONF_DIR` (may be set in properties) | Hadoop client configuration for HDFS writes. | Inside property file or `/etc/hadoop/conf`. |
| `LOG_DIR` (derived) | Directory for `MoveNotification.log`. | `/app/hadoop_users/MNAAS/MNAASCronLogs/` (hard‑coded in script). |

If any of these are missing, the script will fail at launch; a pre‑flight validation step is advisable.

---

## 8. Suggested Improvements (TODO)

1. **Add pre‑flight validation** – before invoking Java, check existence and readability of the property file and jar, and verify write permission on the log directory. Exit with a clear error code and message if any check fails.

2. **Integrate with a process supervisor** – wrap the launch in a `systemd` unit (or `supervisord`) to guarantee automatic restart, enable health‑checking, and simplify log rotation via `StandardOutput=append:/path/to/log`.

---