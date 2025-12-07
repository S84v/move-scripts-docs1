# Summary
The `log4j.properties` file configures Log4j for the **AsyncNotfHandler** component of the mediation‑addon‑notification service. It sets the root logger to `INFO` level and attaches two appenders: a daily‑rolling file appender (`File`) that writes to `${logfile.name}` and a console appender (`stdout`). This enables persistent log files for audit and real‑time console output for troubleshooting in the telecom production move system.

# Key Components
- **log4j.rootLogger** – Defines the root logger level (`INFO`) and the attached appenders (`File`, `stdout`).  
- **log4j.appender.File** – `org.apache.log4j.DailyRollingFileAppender` that creates a new log file each day.  
  - `File` – Path resolved from the `${logfile.name}` variable.  
  - `layout` – `PatternLayout` with conversion pattern `[%-5p] %d{ISO8601} %c - %m%n`.  
- **log4j.appender.stdout** – `org.apache.log4j.ConsoleAppender` that writes to `System.out`.  
  - `layout` – Same `PatternLayout` as the file appender.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| `${logfile.name}` (system property) | Input | Resolved at JVM startup; determines the physical log file location. |
| Log statements from Java classes (e.g., `org.apache.log4j.Logger`) | Input | Routed to Log4j based on logger hierarchy. |
| File appender (`File`) | Output | Writes formatted log entries to the daily‑rolled file. |
| Console appender (`stdout`) | Output | Writes formatted log entries to standard output (captured by container/OS logs). |
| No external services, DBs, or queues are directly involved. |

# Integrations
- **AsyncNotfHandler Java code** – Calls `Logger.getLogger(<class>)` which inherits the root logger configuration defined here.  
- **Startup scripts / JVM launch** – Must supply the system property `logfile.name` (e.g., `-Dlogfile.name=/var/log/asyncnotfhandler.log`).  
- **Container / orchestration** – Console output (`stdout`) is typically collected by Docker/Kubernetes logging drivers or host syslog.  
- **Monitoring tools** – File appender can be tailed by log aggregation platforms (e.g., Splunk, ELK).

# Operational Risks
- **Missing `logfile.name` property** → Log4j defaults to a relative file path, potentially filling the working directory. *Mitigation*: Enforce property via startup wrapper script; fail fast if undefined.  
- **Unbounded log growth** → DailyRollingFileAppender does not purge old files. *Mitigation*: Implement external log rotation/compression or switch to `RollingFileAppender` with size limits.  
- **Insufficient log level** → `INFO` may omit debug details needed for root‑cause analysis. *Mitigation*: Allow dynamic level change via JMX or environment variable.  
- **Performance impact** → Synchronous file writes on high‑throughput paths. *Mitigation*: Use async appenders (e.g., `AsyncAppender`) if latency becomes measurable.

# Usage
```bash
# Example JVM launch (Linux)
JAVA_OPTS="-Dlogfile.name=/opt/mnaas/logs/asyncnotfhandler.log"
exec java $JAVA_OPTS -cp lib/* com.mnaas.AsyncNotfHandlerMain
```
- To debug logging configuration, enable Log4j internal debugging:
```bash
-Dlog4j.debug=true
```
- To view live logs:
```bash
tail -f /opt/mnaas/logs/asyncnotfhandler.log
```

# configuration
- **System Property**: `logfile.name` – absolute path for the daily‑rolling log file.  
- **External Files**: None referenced directly; relies on the property above.  
- **Environment Variables**: Not required by this file, but deployment scripts may map them to `logfile.name`.

# Improvements
1. **Add Log Rotation Retention** – Replace `DailyRollingFileAppender` with `RollingFileAppender` configured with `MaxBackupIndex` and `MaxFileSize` to limit disk usage.  
2. **Enable Runtime Level Adjustment** – Add a JMX bean or expose a REST endpoint to change logger levels without redeploy.  