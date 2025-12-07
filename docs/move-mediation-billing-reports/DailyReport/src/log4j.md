# Summary
`log4j.properties` configures Log4j for the Move‑Mediation‑Billing‑Reports daily job. It defines a root logger at INFO level that writes timestamped, level‑prefixed messages to a daily‑rolling file (path supplied by `${logfile.name}`) and to the console (STDOUT). This enables audit‑trail creation for DAO execution, shell script orchestration, and error notification.

# Key Components
- **log4j.rootLogger** – sets global logging level (INFO) and attaches two appenders: `File` and `stdout`.
- **File appender (`org.apache.log4j.DailyRollingFileAppender`)** – writes logs to `${logfile.name}`; rolls over each day.
- **File layout (`PatternLayout`)** – format `[%-5p] %d{ISO8601} %c - %m%n`.
- **Console appender (`org.apache.log4j.ConsoleAppender`)** – streams identical formatted logs to `System.out`.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input** | System property `logfile.name` (or environment variable) | Determines absolute path of the daily log file. |
| **Output** | `DailyRollingFileAppender` → file system | Persists all INFO‑level and above events per day. |
| **Output** | `ConsoleAppender` → STDOUT | Mirrors log entries to the process console (useful for cron or interactive runs). |
| **Side‑effects** | Log file creation, daily rollover, file descriptor usage. |
| **External services** | None (purely local I/O). |

# Integrations
- **Java classes** in `move-mediation-billing-reports.DailyReport` (DAO, service, and utility classes) obtain a logger via `org.apache.log4j.Logger.getLogger(ClassName.class)`. The configuration is automatically applied at JVM startup.
- **Shell scripts** that launch the Java job inherit the `logfile.name` property via `-Dlogfile.name=/path/to/logfile.log` and thus route Java logs to the same file used by other job components.
- **Monitoring/Alerting** tools may tail the generated log file for health checks or error detection.

# Operational Risks
- **Unbounded log growth** – daily files may accumulate if old files are not purged. *Mitigation*: implement a log‑retention script (e.g., delete files > 30 days) or configure `log4j.appender.File.MaxBackupIndex`.
- **Missing `logfile.name`** – logger falls back to a literal `${logfile.name}` path, causing I/O errors. *Mitigation*: enforce presence via startup wrapper that validates the property.
- **File permission issues** – insufficient write rights cause job failure. *Mitigation*: provision the log directory with appropriate user/group ownership before deployment.
- **Concurrent job instances** – multiple processes writing to the same file can cause contention. *Mitigation*: include a unique identifier (e.g., date‑time or PID) in the log filename.

# Usage
```bash
# Set log file location (environment or JVM property)
export LOGFILE=/var/log/mnaas/daily_report_$(date +%F).log

# Run the Java daily job with Log4j configuration
java -Dlogfile.name=${LOGFILE} \
     -Dlog4j.configuration=file:/opt/mnaas/DailyReport/src/log4j.properties \
     -cp "/opt/mnaas/DailyReport/lib/*:/opt/mnaas/DailyReport/bin" \
     com.mnaas.dailyreport.Main
```
*To debug*: change `log4j.rootLogger` to `DEBUG` and re‑run; verify console output and file creation.

# configuration
- **Environment variable / JVM property**: `logfile.name` – absolute path for the daily rolling log file.
- **Referenced config file**: `log4j.properties` itself (loaded via `-Dlog4j.configuration` or default classpath location).
- **Dependent property files**: `MOVEDAO.properties`, `MNAAS_ShellScript.properties` (not directly used by Log4j but part of the same job).

# Improvements
1. **Add log rotation retention** – configure `log4j.appender.File.MaxBackupIndex` or switch to `RollingFileAppender` with size‑based rollover to bound disk usage.
2. **Introduce an ERROR‑level file appender** – separate critical failures into a dedicated `error.log` for faster alerting and easier triage.