# Summary
The `log4j.properties` file configures Log4j for the Move‑Geneva mediation batch. It defines a root logger at INFO level with two appenders: a daily‑rolling file appender (`File`) and a console appender (`stdout`). Log messages are formatted with a standard pattern and written to the file path supplied by the `${logfile.name}` system property.

# Key Components
- **log4j.rootLogger** – Sets the root logging level (`INFO`) and attaches the `File` and `stdout` appenders.  
- **log4j.appender.File** – `org.apache.log4j.DailyRollingFileAppender` that creates a new log file each day.  
  - `File` – Destination path resolved from `${logfile.name}`.  
  - `layout` – `PatternLayout` with conversion pattern `[%-5p] %d{ISO8601} %c - %m%n`.  
- **log4j.appender.stdout** – `org.apache.log4j.ConsoleAppender` directing logs to `System.out`.  
  - `layout` – Same `PatternLayout` as the file appender.

# Data Flow
| Element | Description |
|---------|-------------|
| **Input** | System property `logfile.name` supplied at JVM start (e.g., `-Dlogfile.name=/var/log/move/move.log`). |
| **Processing** | Log4j evaluates the configuration at initialization, creates the two appenders, and routes all logger events at INFO or higher to both destinations. |
| **Output** | Daily‑rolled log files on the filesystem; real‑time log lines on the console (STDOUT). |
| **Side Effects** | File creation, file rotation, and console I/O. No external services, DBs, or queues are invoked. |

# Integrations
- **Java services** (`MailService`, `MediationFetchService`, `BillingUtils`, etc.) use `org.apache.log4j.Logger` instances; they inherit this root configuration unless overridden.  
- **Batch launcher scripts** (e.g., shell scripts that start the mediation job) must set `logfile.name` before invoking the Java process.  
- **Monitoring/ops tools** may tail the console output or ingest the daily log files for alerting.

# Operational Risks
- **Missing `logfile.name`** → Log4j defaults to a relative file path, potentially filling the working directory and causing disk‑space exhaustion. *Mitigation*: enforce the JVM argument `-Dlogfile.name=…` in launch scripts; add a startup check.  
- **Unbounded log growth** → DailyRollingFileAppender does not purge old files. *Mitigation*: schedule a log‑rotation cleanup (e.g., cron) or switch to `RollingFileAppender` with size limits.  
- **Insufficient log level** → INFO may omit debug details needed for troubleshooting. *Mitigation*: allow overriding the root level via a system property (e.g., `-Dlog4j.rootLogger=DEBUG,File,stdout`).  

# Usage
```bash
# Example launch command
java -Dlogfile.name=/opt/move/logs/move-${HOSTNAME}.log \
     -Dlog4j.debug=true \
     -cp move-mediation-billing.jar com.tcl.move.Main
```
- Verify log file creation: `ls -l /opt/move/logs/`.  
- Tail live console output: `tail -f /opt/move/logs/move-${HOSTNAME}.log`.  

# Configuration
- **Environment / JVM properties**  
  - `logfile.name` – Absolute path for the daily‑rolling log file.  
- **File** – `move-mediation-billing-reports/DailyReport/bin/log4j.properties`.  
- No external configuration files referenced.

# Improvements
1. **Add log retention policy** – Replace `DailyRollingFileAppender` with `RollingFileAppender` configured with `MaxBackupIndex` or implement a log‑cleanup script to delete files older than a configurable retention period.  
2. **Externalize log level** – Introduce a placeholder `${log.level}` with a default of `INFO` to allow runtime adjustment without modifying the properties file. Example: `log4j.rootLogger=${log.level},File,stdout`.  