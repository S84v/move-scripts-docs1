# Summary
The `log4j.properties` file configures Log4j 1.x for the **MoveGenevaInterface** module. It defines a root logger at INFO level with two appenders: a daily‑rolling file appender (`File`) that writes to `${logfile.name}` and a console appender (`stdout`). In production, it controls how operational logs are formatted, persisted, and streamed to the console for monitoring and troubleshooting.

# Key Components
- **log4j.rootLogger** – Sets global log level (INFO) and attaches `File` and `stdout` appenders.  
- **log4j.appender.File** – `org.apache.log4j.DailyRollingFileAppender` writes logs to a file that rolls over daily.  
  - `File` property: `${logfile.name}` (runtime variable).  
  - `layout`: `org.apache.log4j.PatternLayout` with pattern `[%‑5p] %d{ISO8601} %c - %m%n`.  
- **log4j.appender.stdout** – `org.apache.log4j.ConsoleAppender` streams logs to `System.out`.  
  - Same pattern layout as file appender.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| Application code → Log4j API | Input | Calls to `Logger.info/debug/...` generate log events. |
| Log4j → DailyRollingFileAppender | Output | Writes formatted log lines to `${logfile.name}`; file rotates each day. |
| Log4j → ConsoleAppender | Output | Writes same formatted lines to standard output (captured by container/OS logs). |
| External services | None | No direct DB, queue, or network interaction. |
| Side effects | Disk I/O, stdout stream | Persistent log files and console output for monitoring tools. |

# Integrations
- **Runtime environment** supplies `${logfile.name}` (typically via JVM system property `logfile.name`).  
- **Process supervisors / log aggregators** (e.g., `systemd`, Docker, Splunk) consume `stdout` stream.  
- **File rotation** interacts with OS file system; external log rotation or archival scripts may act on rolled files.  
- No direct integration with other application modules; logging is cross‑cutting.

# Operational Risks
- **Missing `logfile.name` property** → Log4j defaults to a literal `${logfile.name}` file, causing I/O errors. *Mitigation*: enforce JVM `-Dlogfile.name=/var/log/movegeneva/interface.log`.  
- **Unbounded disk growth** if daily files are not purged. *Mitigation*: schedule log cleanup or configure external log management.  
- **Log level too verbose** (INFO may be excessive in high‑throughput environments). *Mitigation*: adjust to WARN/ERROR in production via property override.  
- **Log4j 1.x end‑of‑life** → potential security vulnerabilities. *Mitigation*: migrate to Log4j 2.x or apply security patches.

# Usage
```bash
# Start the service with explicit log file location
java -Dlogfile.name=/var/log/movegeneva/interface.log \
     -Dlog4j.configuration=file:/path/to/log4j.properties \
     -jar movegeneva.jar
```
- To debug logging configuration, set `log4j.debug=true` as a JVM argument.  
- Console output can be observed directly or via `journalctl -u <service>` / Docker logs.

# configuration
- **Environment variable / JVM property**: `logfile.name` – absolute path for the daily rolling log file.  
- **External file**: `log4j.properties` (this file) – must be on the classpath or referenced via `-Dlog4j.configuration`.  
- No additional configuration files referenced.

# Improvements
1. **Migrate to Log4j 2.x** – gain asynchronous logging, improved performance, and active security support.  
2. **Externalize log level** – allow dynamic level changes via a separate `log4j-level.properties` or JMX, reducing need for redeploy.