**File:** `move-mediation-scripts/bin/startBalMoveConsumer.sh`

---

## 1. High‑Level Summary
`startBalMoveConsumer.sh` is a thin wrapper that launches the **Balance Notification consumer** Java process (`BalNotification.jar`). It loads a shared properties file (`MNAAS_ShellScript.properties`) that supplies environment‑specific configuration (e.g., queue endpoints, DB credentials, file locations) and then starts the consumer in *Consumer* mode, directing its runtime log to `MoveBalNotification.log`. In production the script is typically scheduled via cron to keep the balance‑notification ingestion pipeline continuously running.

---

## 2. Key Components & Responsibilities  

| Component | Type | Responsibility |
|-----------|------|----------------|
| `MNAAS_ShellScript.properties` | Configuration file (sourced by the script) | Provides all runtime parameters required by the Java consumer (queue names, DB URLs, authentication, retry limits, etc.). |
| `BalNotification.jar` | Java executable (contains `BalNotification` main class) | Implements the **Balance Notification consumer**: connects to the source queue/topic, reads balance‑change messages, performs any required transformation, and persists results to downstream systems (DB tables, files, or other services). |
| `startBalMoveConsumer.sh` | Bash wrapper script | Sources the shared properties, invokes the Java consumer with the correct arguments, and redirects stdout/stderr to a dedicated log file. |
| `MoveBalNotification.log` | Log file (under `/app/hadoop_users/MNAAS/MNAASCronLogs/`) | Captures the consumer’s operational log (startup, processing stats, errors). |

*Note:* The Java consumer likely contains classes such as `BalNotificationConsumer`, `MessageProcessor`, and `DBWriter`, but those are encapsulated inside the JAR and not visible from the shell script.

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • The properties file `MNAAS_ShellScript.properties` (absolute path).<br>• Runtime arguments passed to the JAR: `Consumer` (mode flag) and the same properties file path.<br>• External services referenced inside the properties (e.g., Kafka/ActiveMQ broker, Oracle/MySQL DB, SFTP endpoints). |
| **Outputs** | • Log file `MoveBalNotification.log` (append mode).<br>• Side‑effects performed by the Java consumer: insertion/updates to balance‑related tables, possibly file writes or API calls. |
| **Side Effects** | • Network I/O to message broker and downstream systems.<br>• Database writes (balance notifications).<br>• Potential file system writes (e.g., audit files). |
| **Assumptions** | • Java 1.8+ is installed and `java` is on the PATH.<br>• The properties file exists, is readable, and contains valid values.<br>• The JAR and log directory have appropriate execute/read/write permissions for the user running the script (typically a Hadoop service account).<br>• Required external services (message broker, DB) are reachable from the host. |

---

## 4. Integration Points & Call Graph  

```
Cron (or manual) --> startBalMoveConsumer.sh --> BalNotification.jar (Consumer mode)
                                 |
                                 +--> reads MNAAS_ShellScript.properties
                                 |
                                 +--> connects to Queue (e.g., Kafka topic "BAL_NOTIF")
                                 |
                                 +--> writes to DB tables (BALANCE_NOTIF, BAL_LOG)
                                 |
                                 +--> logs to MoveBalNotification.log
```

*Connections to other scripts*  

- **`startAsyncMoveConsumer.sh`** – a sibling wrapper that launches a different consumer (likely for async events). Both scripts share the same properties file, so changes there affect both pipelines.  
- **`MNAAS_move_files_from_staging.sh`**, **`MNAAS_Weekly_KYC_Feed_Loading.sh`**, etc. – upstream jobs that may populate the message broker or staging tables that the balance consumer later reads.  

Thus, `startBalMoveConsumer.sh` is part of a broader “move” ecosystem where data is staged, transformed, and finally consumed by specialized Java loaders.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing/Corrupt properties file** | Consumer fails to start or uses default (wrong) values. | Add a pre‑flight check: verify file existence and non‑empty before invoking Java; exit with clear error code. |
| **Java process crashes (OOM, uncaught exception)** | No balance notifications processed → data loss/backlog. | Monitor `MoveBalNotification.log` and process PID; configure a watchdog (systemd service or cron “restart if not running”). |
| **Log file permission/rotation issues** | Log growth consumes disk; new entries may be dropped. | Ensure log directory is owned by the service account; integrate with logrotate (size‑based, keep 7 days). |
| **Queue connectivity loss** | Consumer stalls, messages accumulate. | Implement retry/back‑off inside the Java consumer (should already exist); alert on prolonged “no messages processed” condition. |
| **Database schema changes** | Inserts/updates fail, causing transaction rollbacks. | Version‑control DB schema; run compatibility tests when schema evolves; add graceful handling in Java code. |

---

## 6. Running & Debugging the Script  

### Normal Execution (cron)

```bash
# Example crontab entry (runs every 5 minutes)
*/5 * * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/startBalMoveConsumer.sh >> /dev/null 2>&1
```

### Manual Run (for testing)

```bash
# Switch to the service account that owns the files
sudo -u hadoop_user bash

# Execute with verbose output
set -x
/app/hadoop_users/MNAAS/move-mediation-scripts/bin/startBalMoveConsumer.sh
echo $?   # should be 0 on success
```

### Debug Steps

1. **Verify properties**  
   ```bash
   cat /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   ```
2. **Check Java version**  
   ```bash
   java -version
   ```
3. **Run the JAR directly** (bypass the wrapper) to see immediate console output:  
   ```bash
   java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/BalNotification.jar Consumer \
        /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   ```
4. **Tail the log** while the process runs:  
   ```bash
   tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/MoveBalNotification.log
   ```
5. **Check process status**  
   ```bash
   ps -ef | grep BalNotification.jar
   ```

If the process exits with a non‑zero status, inspect the last lines of the log for stack traces or configuration errors.

---

## 7. External Config / Environment Dependencies  

| Item | Location | Usage |
|------|----------|-------|
| `MNAAS_ShellScript.properties` | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` | Loaded by the script (`.` source) and passed to the Java consumer; contains queue URLs, DB credentials, retry limits, etc. |
| `BalNotification.jar` | `/app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/` | Executable containing the consumer logic. |
| `MoveBalNotification.log` | `/app/hadoop_users/MNAAS/MNAASCronLogs/` | Destination for consumer stdout/stderr. |
| `JAVA_HOME` (optional) | Environment | Determines which JRE is used; if not set, the system `java` on PATH is used. |
| System user permissions | OS level | The user running the script must have read access to the properties file, execute permission on the JAR, and write permission on the log directory. |

---

## 8. Suggested Improvements (TODO)

1. **Add robust error handling** – after the `java -jar` call, capture the exit code and write a concise “SUCCESS/FAILURE” entry to a health‑check file or monitoring system (e.g., `/var/run/bal_consumer.status`).  
2. **Parameterize log location** – accept an optional third argument for the log file path, allowing the same script to be reused in different environments (dev/test/prod) without hard‑coding the path.  

---