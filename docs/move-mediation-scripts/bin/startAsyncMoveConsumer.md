**File:** `move-mediation-scripts\bin\startAsyncMoveConsumer.sh`

---

## 1. High‑Level Summary
`startAsyncMoveConsumer.sh` is a thin wrapper that launches the **AsyncNotification** Java consumer process for the MNAAS (Move‑Network‑As‑a‑Service) data‑move platform. It loads a shared property file, then starts `AsyncNotification.jar` in *Consumer* mode, passing the same property file and a dedicated log file. The consumer continuously reads asynchronous notification messages (e.g., from a message broker or file‑drop directory) and triggers downstream move‑orchestration actions such as file staging, database updates, or API calls.

---

## 2. Key Components & Responsibilities  

| Component | Type | Responsibility |
|-----------|------|----------------|
| `startAsyncMoveConsumer.sh` | Bash wrapper | Sources common properties, launches the Java consumer, redirects output to a rotating log. |
| `MNAAS_ShellScript.properties` | Property file (key‑value) | Holds environment‑specific configuration: broker endpoints, DB credentials, file paths, thread counts, etc. |
| `AsyncNotification.jar` | Java executable | Contains the **Consumer** class that connects to the async notification source, parses messages, and invokes the appropriate move‑pipeline services. |
| `MoveAsyncNotification.log` | Log file | Captures stdout/stderr from the Java process for audit, troubleshooting, and monitoring. |

*Note:* The internal Java classes (e.g., `Consumer`, `NotificationHandler`, `MoveOrchestrator`) are not visible here but are invoked by the jar.

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` (mandatory) <br>• Implicit environment variables (e.g., `JAVA_HOME`, `PATH`) <br>• Asynchronous notification source defined in the properties (e.g., Kafka topic, SFTP drop folder, DB table). |
| **Outputs** | • `/app/hadoop_users/MNAAS/MNAASCronLogs/MoveAsyncNotification.log` (text log) <br>• Side‑effects performed by the Java consumer: file moves, DB inserts/updates, API calls, metric emission. |
| **Side Effects** | • Creation/modification of files in staging/production directories. <br>• Updates to MNAAS relational tables (e.g., `move_job`, `notification_audit`). <br>• Calls to downstream REST/SOAP services (e.g., provisioning, billing). |
| **Assumptions** | • Java 8+ runtime is installed and reachable via `java`. <br>• The property file exists, is readable, and contains valid values. <br>• The jar and log directories have appropriate permissions for the executing user. <br>• The async source (broker, SFTP, etc.) is reachable and correctly configured. |

---

## 4. Integration Points with Other Scripts / Components  

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| **`MNAAS_move_files_from_staging.sh`** | Likely produces files that generate async notifications consumed by this jar. |
| **`MNAAS_Weekly_KYC_Feed_Loading.sh`** | May emit KYC‑related notifications that the consumer processes. |
| **`api_med_subs_activity_loading.sh`** | Could be a downstream consumer triggered by messages processed here. |
| **Cron / Scheduler** | The script is typically invoked from a cron entry (e.g., `MNAASCronLogs`) to ensure the consumer is always running or restarted on failure. |
| **Message Broker / SFTP** | Defined in `MNAAS_ShellScript.properties`; the consumer connects here to pull notifications. |
| **Database / API Services** | The consumer uses credentials from the property file to persist results or call external services. |

*Typical flow:* Producer scripts place files or publish messages → notification is queued → `startAsyncMoveConsumer.sh` runs the consumer → consumer reads the notification, performs the required move/orchestration, logs the outcome.

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed property file** | Consumer fails to start, causing data‑move backlog. | Validate file existence & syntax before launching; add a pre‑flight check in the script. |
| **Java process crash / OOM** | Loss of processing, possible duplicate handling on restart. | Enable JVM options for memory limits (`-Xmx`), monitor process health, configure a watchdog (systemd, supervisord). |
| **Unbounded log growth** | Disk exhaustion, loss of recent logs. | Rotate `MoveAsyncNotification.log` via `logrotate` or implement size‑based rollover inside the Java app. |
| **Network / broker outage** | Consumer stalls, message backlog. | Implement retry/back‑off in the Java consumer; alert on prolonged connection failures. |
| **Permission issues on staging directories** | Files cannot be moved, causing pipeline failures. | Ensure the executing user has read/write rights; audit permissions during deployment. |
| **Duplicate processing** | Data inconsistency. | Verify idempotent handling in the Java consumer; use message offsets or unique IDs. |

---

## 6. Running & Debugging the Script  

1. **Standard execution (as scheduled):**  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_ShellScript.properties && \
   /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/AsyncNotification.jar Consumer \
   /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties \
   /app/hadoop_users/MNAASCronLogs/MoveAsyncNotification.log
   ```

2. **Manual start (interactive):**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars
   java -jar AsyncNotification.jar Consumer \
        /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties \
        /tmp/MoveAsyncNotification.log
   ```

3. **Enable shell tracing for troubleshooting:**  
   ```bash
   set -x   # add at top of script or run: bash -x startAsyncMoveConsumer.sh
   ```

4. **Check process:**  
   ```bash
   ps -ef | grep AsyncNotification.jar
   ```

5. **Inspect logs:**  
   ```bash
   tail -f /app/hadoop_users/MNAASCronLogs/MoveAsyncNotification.log
   ```

6. **Validate properties:**  
   ```bash
   grep -v '^#' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   ```

7. **Restart on failure:**  
   Wrap the call in a simple loop or use a service manager (systemd) to auto‑restart on non‑zero exit.

---

## 7. External Configuration & Environment Dependencies  

| Item | Purpose | Location |
|------|---------|----------|
| `MNAAS_ShellScript.properties` | Central configuration (broker URLs, DB creds, thread pool sizes, file paths). | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` |
| `JAVA_HOME` (optional) | Determines which Java runtime is used. | Environment variable; if unset, script relies on `java` in `PATH`. |
| `PATH` | Must include the Java binary directory. | System environment. |
| `AsyncNotification.jar` | Executable containing the consumer logic. | `/app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/` |
| Log directory `MNAASCronLogs` | Destination for runtime logs. | `/app/hadoop_users/MNAASCronLogs/` |

If any of these paths change, the script must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Add a health‑check wrapper** – Create a systemd unit or supervisord config that monitors the Java process, restarts on failure, and exposes a simple HTTP/CLI health endpoint.
2. **Implement log rotation** – Integrate `logrotate` or configure the Java logger to roll over `MoveAsyncNotification.log` daily or when it exceeds a configurable size, preventing disk‑full incidents.