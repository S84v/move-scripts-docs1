# Summary
`log4j.properties` configures Log4j for the **Move Mediation Notification Handler** component. It defines a root logger that writes INFO‑level (and higher) messages to a daily‑rolling file appender and to the console (stdout). The file appender path is supplied via the `${logfile.name}` system property at runtime.

# Key Components
- **log4j.rootLogger** – Sets the root logger level to `INFO` and attaches two appenders: `File` and `stdout`.
- **File appender (`org.apache.log4j.DailyRollingFileAppender`)** – Writes logs to a file that rolls over each day.
  - `File` – Destination file name (`${logfile.name}`).
  - `layout` – `PatternLayout` with conversion pattern `[%-5p] %d{ISO8601} %c - %m%n`.
- **Console appender (`org.apache.log4j.ConsoleAppender`)** – Streams logs to `System.out`.
  - Same `PatternLayout` as the file appender.

# Data Flow
| Element | Direction | Details |
|---------|-----------|---------|
| **Input** | Runtime system property | `${logfile.name}` – absolute or relative path for the daily log file. |
| **Output** | File system | Daily‑rolled log file written to the path defined by `${logfile.name}`. |
| **Side‑effects** | Console | Log lines emitted to standard output, captured by container/host logs. |
| **External services** | None (pure logging). |
| **DB / Queues** | None. |

# Integrations
- **Java classes** in `move-mediation-common-notification\NotfHandler` that obtain a `org.apache.log4j.Logger` instance (e.g., `Logger.getLogger(ClassName.class)`) automatically inherit this configuration.
- **Deployment scripts** that set `-Dlogfile.name=/var/log/move/notification.log` (or similar) when launching the JVM.
- **Monitoring/Log aggregation** tools (e.g., ELK, Splunk) ingest the generated log files or capture stdout.

# Operational Risks
- **Missing `logfile.name` property** → Log4j defaults to a file named `null`, causing I/O errors. *Mitigation*: enforce property via startup script or provide a default value in the properties file.
- **Unbounded log growth** → DailyRollingFileAppender does not purge old files. *Mitigation*: schedule a log‑rotation cleanup (cron) or switch to `RollingFileAppender` with size‑based retention.
- **Insufficient log level** → INFO may omit debug information needed for troubleshooting. *Mitigation*: allow dynamic level change via JMX or external config for specific packages.

# Usage
```bash
# Example JVM launch
java -Dlogfile.name=/opt/move/logs/notification.log \
     -cp move-mediation-common-notification.jar \
     com.tcl.move.notification.Main
```
- Verify log file creation in `/opt/move/logs/`.
- Tail console output: `tail -f /opt/move/logs/notification.log`.

# Configuration
- **Environment variable / system property**: `logfile.name` – required; points to the target log file.
- **File**: `log4j.properties` located in the classpath (`move-mediation-common-notification\NotfHandler\bin`).

# Improvements
1. **Add log retention** – Replace `DailyRollingFileAppender` with `RollingFileAppender` configured with `MaxBackupIndex` to automatically delete old logs.
2. **Externalize log level** – Introduce a property (e.g., `log4j.rootLogger=INFO, File, stdout`) that can be overridden at runtime to enable DEBUG for specific packages without code changes.