# Summary
`AsyncNotificationHandler` is the entry‑point for the Move‑Mediation add‑on notification service. It loads runtime properties, configures logging, and either (a) starts a Kafka‑style consumer to process asynchronous notifications in real time or (b) reads a static re‑process file, parses each JSON record, sends alert mail on parse failures, and forwards successfully parsed `AddonRecord` objects to `NotificationService` for downstream handling.

# Key Components
- **`main(String[] args)`** – parses command‑line arguments (`method`, `propertyFile`, `logFile`), loads properties into `System`, initializes logger, and dispatches to `start()` or `readFile()`.
- **`readFile()`** –  
  - Opens hard‑coded file `/app/hadoop_users/MNAAS/Notification_ReProcess_Files/asyncNot.txt`.  
  - Iterates line‑by‑line, invoking `JSONParseUtils.parseAsyncJSON`.  
  - On `ParseException`, builds an alert message and calls `mailService.sendAlertMail`.  
  - Accumulates valid `AddonRecord` objects and passes the list to `notificationService.processAsyncRequest`.
- **`start()`** –  
  - Retrieves singleton `NotificationConsumer` via `NotificationConsumer.getInstance()`.  
  - Calls `runConsumer()` to begin asynchronous consumption.  
  - Sleeps 5 minutes, then invokes `service.shutdown()`.
- **`getStackTrace(Exception)`** – utility to convert an exception stack trace to a string for logging.
- **Member fields** – `static Logger logger`, `MailService mailService`.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Startup | CLI args (`method`, `propertyFile`, `logFile`) | Load properties → set `System` properties → configure Log4j | Logger ready, environment configured |
| File mode (`readFile`) | Text file lines (JSON) | `JSONParseUtils.parseAsyncJSON` → `AddonRecord` objects | `NotificationService.processAsyncRequest(List<AddonRecord>)` |
| File mode – error path | `ParseException` | Build alert body → `MailService.sendAlertMail` | Alert email sent |
| Consumer mode (`start`) | External message broker (implementation hidden in `NotificationConsumer`) | `runConsumer()` processes messages | Business logic executed inside consumer; later `shutdown()` stops it |
| Logging | All catch blocks | Log4j writes to file defined by `logFile` | Persistent log entries |

External services:
- **Mail server** via `MailService.sendAlertMail`.
- **Message broker** accessed inside `NotificationConsumer`.
- **Database / downstream systems** accessed indirectly by `NotificationService.processAsyncRequest` (not shown).

# Integrations
- **`Constants`** – static fields `logFile` and `propertyFile` are populated for global use.
- **`MailService`** – used for alert notifications on parse failures.
- **`NotificationConsumer`** – singleton consumer that likely connects to Kafka/RabbitMQ (implementation not in this file).
- **`NotificationService`** – receives parsed records for further business processing (e.g., DB writes, downstream API calls).
- **`JSONParseUtils`** – parses inbound JSON payloads into `AddonRecord` DTOs.
- **`com.tcl.move.exceptions`** – custom checked exceptions (`MailException`, `ParseException`, `DatabaseException`) used for error categorisation.

# Operational Risks
- **Hard‑coded file path** – may not exist or have insufficient permissions; leads to silent failure. *Mitigation*: externalize path via property.
- **Fixed 5‑minute consumer run window** – may truncate processing under load. *Mitigation*: make shutdown timeout configurable or use graceful shutdown signals.
- **Broad `catch (Exception e)` in `readFile`** – masks specific failures and may hide resource leaks. *Mitigation*: catch specific exceptions and ensure `finally` block closes streams.
- **No validation of CLI arguments** – `ArrayIndexOutOfBoundsException` if args missing. *Mitigation*: add argument count check with usage message.
- **Potential memory pressure** – accumulating all parsed records in a list before batch processing could exhaust heap for large files. *Mitigation*: stream records to `NotificationService` in smaller batches.

# Usage
```bash
# Production invocation
java -cp <classpath> com.tcl.move.main.AsyncNotificationHandler Consumer /opt/mnaas/conf/MNAAS_ShellScript.properties /var/log/mnaas/MoveAsyncNotification.log

# File re‑process mode
java -cp <classpath> com.tcl.move.main.AsyncNotificationHandler File /opt/mnaas/conf/MNAAS_ShellScript.properties /var/log/mnaas/MoveAsyncNotification.log
```
*Debug*: Run with IDE, set breakpoints in `readFile` or `start`, and supply mock property file paths.

# configuration
- **System properties file** – path supplied as second CLI argument; loaded into `System` at startup.
- **Log file** – third CLI argument; also set as Log4j property `logfile.name`.
- **Hard‑coded re‑process file** – `/app/hadoop_users/MNAAS/Notification_ReProcess_Files/asyncNot.txt` (should be overridden via property if needed).
- **Mail server configuration** – expected to be defined in the loaded properties (e.g., SMTP host, port, credentials) used by `MailService`.

# Improvements
1. **Externalize all file paths and timeouts** – replace hard‑coded re‑process file location and consumer sleep duration with configurable properties.
2. **Resource management & error handling** – use try‑with‑resources for `FileReader`/`BufferedReader`, replace generic `catch (Exception)` with specific catches, and ensure proper closure of streams even on parsing errors.