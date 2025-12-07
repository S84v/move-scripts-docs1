# Summary
`BalanceNotificationHandler` is the entry point for the Move mediation balance‑notification batch. It loads runtime properties, configures logging, and either starts the Kafka‑style consumer (`NotificationConsumer`) or processes a re‑process file line‑by‑line, delegating each record to `NotificationService.processBalanceRequest`. The class also provides utility for stack‑trace extraction.

# Key Components
- **public static void main(String[] args)**
  - Parses command‑line arguments: method, property file path, log file path.
  - Sets `Constants.logFile` and `Constants.propertyFile`.
  - Loads properties into `System` properties.
  - Instantiates `BalanceNotificationHandler` and invokes `start()` or `readFile()` based on the method argument.
- **private void start()**
  - Retrieves singleton `NotificationConsumer`.
  - Calls `runConsumer()` to begin message consumption.
  - Sleeps 5 minutes, then calls `shutdown()` on the consumer.
- **private void readFile()**
  - Instantiates `NotificationService`.
  - Reads hard‑coded file `/app/hadoop_users/MNAAS/Notification_ReProcess_Files/balNot.txt`.
  - For each line, invokes `notificationService.processBalanceRequest(line)`.
- **private static String getStackTrace(Exception e)**
  - Converts an exception stack trace to a `String` for logging.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Startup | CLI args (`method`, `propertyFile`, `logFile`) | Load properties → set system properties | Configured environment, logger initialized |
| Consumer mode | Messages from external queue (implementation inside `NotificationConsumer`) | `runConsumer()` processes each message via `NotificationService` (not shown) | Business logic execution, possible DB updates, logs |
| File mode | Lines from `/app/hadoop_users/MNAAS/Notification_ReProcess_Files/balNot.txt` | `processBalanceRequest(line)` per line | Business logic execution, possible DB updates, logs |
| Error handling | Exceptions from property loading, file I/O, consumer start/shutdown | Logged with stack trace | No exception propagation; process continues or terminates after main block |

# Integrations
- **`com.tcl.move.constants.Constants`** – receives `logFile` and `propertyFile` values.
- **`org.apache.log4j.Logger`** – logging framework; log file path supplied via system property `logfile.name`.
- **`com.tcl.move.service.NotificationConsumer`** – singleton consumer handling message queue (e.g., Kafka, JMS). Not shown but invoked.
- **`com.tcl.move.service.NotificationService`** – business service that processes each balance request.
- **External file system** – reads re‑process file from a fixed HDFS‑like path.
- **System properties** – made available to downstream components for configuration (DB URLs, credentials, etc.).

# Operational Risks
- **Hard‑coded file path** in `readFile()` limits portability and may cause FileNotFound errors in non‑standard environments. *Mitigation*: externalize path via property.
- **Fixed 5‑minute sleep before shutdown** may lead to premature termination of in‑flight messages under variable load. *Mitigation*: make shutdown grace period configurable or use consumer’s own termination signals.
- **Broad `catch (Exception e)`** masks specific failures and prevents targeted alerts. *Mitigation*: catch and log specific exception types (e.g., `IOException`, `DatabaseException`).
- **No validation of CLI arguments**; missing or malformed args cause `ArrayIndexOutOfBoundsException`. *Mitigation*: add argument count check and usage help.
- **Logger may be null if property loading fails before initialization**, leading to NPE in catch blocks. *Mitigation*: initialize logger early or use fallback logger.

# Usage
```bash
# Production invocation
java -cp <classpath> com.tcl.move.main.BalanceNotificationHandler Consumer /opt/mnaas/conf/MNAAS_ShellScript.properties /var/log/mnaas/MoveBalNotification.log

# File re‑process mode
java -cp <classpath> com.tcl.move.main.BalanceNotificationHandler File /opt/mnaas/conf/MNAAS_ShellScript.properties /var/log/mnaas/MoveBalNotification.log
```
*Debug*: Run from IDE with the same three arguments; set breakpoints in `start()` or `readFile()`.

# configuration
- **System properties file**: path supplied as second CLI argument; must contain all keys required by downstream services (DB URLs, queue endpoints, etc.).
- **Log file**: path supplied as third CLI argument; also set via system property `logfile.name` for Log4j configuration.
- **Constants**: `Constants.logFile` and `Constants.propertyFile` are populated at runtime.
- **Re‑process file**: currently hard‑coded; should be added to the properties file (e.g., `reprocess.file.path`).

# Improvements
1. **Externalize re‑process file path** and make it configurable via the properties file; replace the hard‑coded string with `System.getProperty("reprocess.file.path")`.
2. **Replace fixed sleep with configurable graceful shutdown**: read a `consumer.shutdown.wait.ms` property and use it to determine the wait time, or implement a listener that triggers shutdown when the consumer reports idle state.