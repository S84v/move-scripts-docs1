**High‑Level Documentation – `move-mediation-scripts/bin/startTestMoveConsumer.sh`**

---

### 1. Purpose (One‑paragraph Summary)
`startTestMoveConsumer.sh` is a lightweight wrapper used to launch the **TestNotif** Java consumer component in a development / test environment. It loads the shared MNAAS property file, then executes `TestNotif.jar` with the `Consumer` mode, passing the same property file and a dedicated log file (`TestNotification.log`). The script is primarily used by developers or test operators to verify that the notification‑consumer pipeline can start, connect to required services (e.g., Kafka, MQ, or internal queues), and write diagnostic output to the test log.

---

### 2. Key Elements

| Element | Type | Responsibility |
|---------|------|----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`** | Shell source | Loads environment variables (e.g., `JAVA_HOME`, `HADOOP_CONF_DIR`, service endpoints, credentials) required by the Java loader. |
| **`java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/TestNotif.jar Consumer …`** | Java invocation | Starts the `TestNotif` application in *Consumer* mode. The JAR contains the main class that subscribes to the test notification source, processes messages, and writes status to the supplied log. |
| **`/app/hadoop_users/MNAAS/MNAASCronLogs/TestNotification.log`** | Log file | Destination for stdout/stderr of the Java process; used for troubleshooting and audit. |

*No functions or additional classes are defined in the shell script itself; the heavy lifting resides in `TestNotif.jar`.*

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_ShellScript.properties` (environment configuration).<br>• Implicit system environment (PATH, JAVA_HOME). |
| **Outputs** | • `TestNotification.log` (text log).<br>• Potential side‑effects produced by the Java consumer (e.g., messages consumed from a queue, database writes, metric updates). |
| **Side Effects** | • Opens network connections to the notification source (Kafka, MQ, REST, etc.) as defined in the properties file.<br>• May create/modify temporary files or database rows depending on the consumer implementation. |
| **Assumptions** | • Java runtime is installed and matches the version used to compile `TestNotif.jar`.<br>• The property file exists and contains all required keys (service URLs, credentials, Hadoop configs).<br>• The user executing the script has write permission to `/app/hadoop_users/MNAAS/MNAASCronLogs/`.<br>• No other instance of the consumer is already bound to the same resources (e.g., exclusive consumer groups). |

---

### 4. Integration Points (How it Connects to the Rest of the System)

| Connection | Description |
|------------|-------------|
| **Property File (`MNAAS_ShellScript.properties`)** | Shared across all MOVE scripts (e.g., `runGBS.sh`, `service_based_charges.sh`). It defines common environment variables and service endpoints, ensuring consistent configuration. |
| **Java Loader JARs (`Loader_Jars/*.jar`)** | The same directory houses other loader JARs used by production scripts (e.g., `BalLoader.jar`, `NotifLoader.jar`). `TestNotif.jar` follows the same command‑line contract (`<mode> <propFile> <logFile>`). |
| **Log Directory (`MNAASCronLogs/`)** | Centralised log location used by many cron‑driven scripts; downstream monitoring tools (Splunk, ELK) ingest these logs for alerting. |
| **Potential Down‑stream Scripts** | While not directly invoked, the consumer may populate staging tables or message queues that are later processed by scripts such as `processNotifBackup.sh` or `runMLNS.sh`. |
| **Cron / Scheduler** | In a test environment, the script may be scheduled via a dedicated cron entry to periodically validate the consumer health. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing / malformed property file** | Consumer fails to start, silent error if not logged. | Add a pre‑run check: `[[ -r $PROP_FILE ]] || { echo "Property file missing"; exit 1; }`. |
| **Java version mismatch** | Runtime `UnsupportedClassVersionError`. | Pin `JAVA_HOME` in the property file and verify with `java -version` before launch. |
| **Log file not writable** | No diagnostic output; operator blind to failures. | Ensure directory permissions; rotate logs or use `touch` to create file with correct ownership. |
| **Uncontrolled consumer instances** | Duplicate consumption, message loss or duplication. | Implement a PID lock file (e.g., `/var/run/startTestMoveConsumer.pid`) and exit if already running. |
| **Network/service endpoint down** | Consumer hangs or retries indefinitely. | Configure reasonable connection timeouts in the Java code; optionally wrap the java call with a timeout utility (`timeout 30m java …`). |
| **No error handling in script** | Non‑zero exit codes are ignored. | Capture `$?` after the java command and exit with that status; optionally email/notify on failure. |

---

### 6. Running & Debugging Guide

1. **Standard Execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars
   /app/hadoop_users/MNAAS/MNAAS_ShellScript/bin/startTestMoveConsumer.sh
   ```

2. **Check Immediate Output**  
   The script itself produces no console output; inspect the log:  
   ```bash
   tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/TestNotification.log
   ```

3. **Debug Mode**  
   Add shell tracing to see the expanded command:  
   ```bash
   set -x
   . /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/TestNotif.jar Consumer \
        /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties \
        /app/hadoop_users/MNAAS/MNAASCronLogs/TestNotification.log
   set +x
   ```

4. **Verify Exit Status**  
   ```bash
   ./startTestMoveConsumer.sh
   echo "Exit code: $?"
   ```

5. **Log Rotation** (if log grows):  
   ```bash
   mv /app/hadoop_users/MNAAS/MNAASCronLogs/TestNotification.log \
      /app/hadoop_users/MNAAS/MNAASCronLogs/TestNotification.log.$(date +%Y%m%d%H%M%S)
   ```

6. **Common Checks**  
   - `ls -l /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`  
   - `java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/TestNotif.jar --help` (if the jar supports it).  

---

### 7. External Configuration & Environment Variables

| File / Variable | Usage |
|-----------------|-------|
| **`MNAAS_ShellScript.properties`** | Supplies `JAVA_HOME`, `HADOOP_CONF_DIR`, service URLs, authentication tokens, and any consumer‑specific parameters (e.g., `notif.consumer.group`, `notif.topic`). |
| **`PATH`, `JAVA_HOME`** (potentially set in the property file) | Required for locating the `java` executable. |
| **`MNAASCronLogs/` directory** | Must exist and be writable; defined in the property file as `LOG_DIR` (if present). |

If any of these files are missing or contain unexpected values, the script will fail silently unless additional checks are added.

---

### 8. Suggested Improvements (TODO)

1. **Add Robust Error Handling** – Verify the existence/readability of the property file and log directory, capture the Java process exit code, and propagate it to the caller (or send an alert).  
2. **Implement Single‑Instance Guard** – Create a PID lock file to prevent concurrent consumer launches, which could cause duplicate processing or resource contention.

---