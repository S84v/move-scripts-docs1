# Summary
The `log4j.properties` file configures Log4j for the **SIMInvLoader** component of the Move mediation pipeline. It establishes a root logger at INFO level that writes formatted log entries to a daily‑rolling file (path supplied by `${logfile.name}`) and to the standard output console. This provides unified, timestamped logging for all Java/Scala processes executed by the SIM inventory loader job in production.

# Key Components
- **log4j.rootLogger** – Sets global logging level to INFO and attaches two appenders: `File` (daily‑rolling file) and `stdout` (console).
- **log4j.appender.File** – `org.apache.log4j.DailyRollingFileAppender` that creates/rotates a log file each day; layout defined by `PatternLayout`.
- **log4j.appender.File.layout** – `PatternLayout` with conversion pattern `[%-5p] %d{ISO8601} %c - %m%n` (level, ISO‑8601 timestamp, logger name, message).
- **log4j.appender.stdout** – `org.apache.log4j.ConsoleAppender` targeting `System.out`; uses the same `PatternLayout` as the file appender.

# Data Flow
- **Inputs**: System property `logfile.name` (provided at JVM start, e.g., `-Dlogfile.name=/var/log/siminvloader.log`).
- **Outputs**: 
  - Daily‑rolled log file written to the path defined by `logfile.name`.
  - Log lines emitted to the process’s standard output stream (captured by the job scheduler or container logs).
- **Side Effects**: File creation/rotation on the host filesystem; console output may be redirected to log aggregation services (e.g., Splunk, ELK).
- **External Services**: None directly; interacts only with the local filesystem and the JVM’s stdout.

# Integrations
- Loaded automatically by Log4j during JVM initialization for any class in the SIMInvLoader classpath.
- Consumed by all DAO, service, and utility classes that use `org.apache.log4j.Logger` (e.g., `SIMInvLoaderJob`, `SIMInvDAO`, `InventoryProcessor`).
- Works in conjunction with `MOVEDAO.properties` and `MNAAS_ShellScript.properties`, which provide the business logic that generates log events.

# Operational Risks
- **Missing `logfile.name`** → Log file fallback to default location or failure to create file; mitigated by enforcing JVM launch script to set `-Dlogfile.name`.
- **Disk space exhaustion** → Unbounded log growth if retention not managed; mitigated by OS logrotate or external log management policies.
- **Incorrect file permissions** → Log writer may lack write access; mitigated by provisioning correct ownership/ACL on log directory.
- **Performance impact** – Synchronous file writes at high volume; mitigated by ensuring adequate I/O bandwidth and considering async appenders if needed.

# Usage
```bash
# Example JVM launch (embedded in the job script)
java -Dlogfile.name=/var/log/siminvloader.log \
     -cp "lib/*:conf" com.move.siminvloader.SIMInvLoaderJob
```
To debug logging level at runtime:
```bash
# Increase to DEBUG for a specific class
log4j.logger.com.move.siminvloader.InventoryProcessor=DEBUG, stdout
```
Add the above line to a temporary `log4j-debug.properties` and pass `-Dlog4j.configuration=log4j-debug.properties`.

# Configuration
- **Environment Variable / JVM Property**: `logfile.name` – absolute path for the daily‑rolling log file.
- **Referenced Config Files**: None; this file is self‑contained but must be on the classpath (`conf/log4j.properties`).

# Improvements
1. **Add Log Rotation Retention** – Switch to `RollingFileAppender` with `MaxBackupIndex` to limit retained files, or integrate with external log‑management rotation.
2. **Introduce Separate Appenders for ERROR** – Add a dedicated `ERROR` file appender to capture stack traces for faster incident triage.