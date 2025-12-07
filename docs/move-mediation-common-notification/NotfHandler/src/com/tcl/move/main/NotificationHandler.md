# Summary
`NotificationHandler` is the entry‑point for the MOVE mediation notification subsystem. It bootstraps runtime configuration, initializes logging, and either launches a Kafka consumer (`NotificationConsumer`) or processes a set of predefined CSV/flat‑file re‑process inputs by delegating to `SwapService` and `NotificationService`. The class is used in production batch jobs and cron‑driven pipelines to replay or ingest notification events.

# Key Components
- **`NotificationHandler.main(String[] args)`** – Parses command‑line arguments (`method`, `propertyFile`, `logFile`), loads system properties, and dispatches to either `start()` or `readFile()`.
- **`readFile()`** – Opens a hard‑coded list of re‑process files, streams each line, and invokes:
  - `SwapService.performSIMSwap`
  - `SwapService.performMSISDNSwap`
  - `NotificationService.processStatusUpdateRequest`
  - `NotificationService.processOtaRequest`
  - `NotificationService.processLocationUpdateRequest` (UPDATE / RAW)
  - `NotificationService.processSubsCreationRequest`
  - `NotificationService.processOCSRequest`
  - `NotificationService.processESimRequest`
  - `NotificationService.processSIMIMEIRequest`
- **`start()`** – Retrieves a singleton `NotificationConsumer`, runs the Kafka consumer loop, sleeps 5 min, then shuts down the consumer.
- **`getStackTrace(Exception)`** – Utility to convert an exception stack trace to a `String` for logging.
- **Static members** – `logger` (Log4j) and references to `Constants.logFile` / `Constants.propertyFile`.

# Data Flow
| Phase | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Init  | Command‑line args (`method`, property file path, log file path) | Loads properties into `System` and sets `Constants` fields; configures Log4j via `logfile.name`. | Populated system properties, initialized logger. |
| Consumer mode (`method=Consumer`) | No external file input | `NotificationConsumer.getInstance().runConsumer()` consumes Kafka messages from the `move‑apinotif‑asynccallback` topic (implementation not shown). | Processed notifications, possible DB writes or downstream calls inside consumer. |
| File mode (`method=File`) | Hard‑coded file paths under `/app/hadoop_users/MNAAS/Notification_ReProcess_Files/` | For each file, reads line‑by‑line, invokes corresponding service method. | Service‑level side‑effects (DB updates, external API calls, logging). |
| Error handling | Exceptions from I/O or service calls | Logged via `logger.error`; stack trace captured by `getStackTrace`. | Error records in log file. |

External services referenced indirectly:
- **`SwapService`** – Business logic for SIM/MSISDN swaps (likely DB updates).
- **`NotificationService`** – Handles status, OTA, location, subscription, OCS, eSIM, IMEI notifications.
- **`NotificationConsumer`** – Kafka consumer (topic, broker details supplied via property file).

# Integrations
- **Property files** – Loaded at startup; provide Kafka broker list, DB connection strings, and other runtime constants consumed by downstream services.
- **Log4j** – Configured via system property `logfile.name`; all components share the same log destination.
- **Kafka** – Consumer instantiated in `start()`; producer side is external (`JsonProducer` class) not invoked here.
- **Database / external APIs** – Invoked inside `SwapService` and `NotificationService` (not visible in this file but required for production processing).
- **File system** – Reads re‑process files from a fixed HDFS‑mounted path; expects UNIX line endings.

# Operational Risks
- **Hard‑coded file paths** – Breaks portability; any path change requires code redeployment. *Mitigation*: externalize via properties.
- **Resource leaks** – `FileReader`/`BufferedReader` are closed manually but not in a `finally` block; exceptions may skip `close()`. *Mitigation*: use try‑with‑resources.
- **Null logger on early failure** – If property loading fails before logger init, `logger` remains `null` causing `NullPointerException`. *Mitigation*: initialize logger before property loading or add null checks.
- **Single‑threaded processing** – Large files processed sequentially; may cause backlog. *Mitigation*: parallelize per file or batch.
- **Unvalidated command‑line args** – Missing or malformed arguments cause `ArrayIndexOutOfBoundsException`. *Mitigation*: add argument validation and usage help.

# Usage
```bash
# Compile (example Maven)
mvn clean package

# Run consumer mode
java -cp target/notfhandler.jar com.tcl.move.main.NotificationHandler Consumer /opt/mnaas/conf/MNAAS_ShellScript.properties /var/log/mnaas/MoveNotification.log

# Run file re‑process mode
java -cp target/notfhandler.jar com.tcl.move.main.NotificationHandler File /opt/mnaas/conf/MNAAS_ShellScript.properties /var/log/mnaas/MoveNotification.log
```
For debugging, attach a remote JVM debugger to the process or run with `-Dlog4j.debug=true` to see Log4j configuration details.

# Configuration
- **Property file** (`args[1]`) – Must contain all keys required by downstream services (Kafka `bootstrap.servers`, DB URLs, authentication tokens, etc.).
- **Log file** (`args[2]`) – Path written to `Constants.logFile` and used by Log4j via `logfile.name`.
- **Constants** – `Constants.propertyFile` and `Constants.logFile` are set at runtime; other constants are read from system properties.
- **Environment** – Assumes access to `/app/hadoop_users/MNAAS/Notification_ReProcess_Files/` directory and appropriate read permissions.

# Improvements
1. **Externalize file locations** – Add properties `reprocess.sim.file`, `reprocess.msisdn.file`, etc., and read them instead of hard‑coding absolute paths.
2. **Modern resource handling** – Refactor `readFile()` to use try