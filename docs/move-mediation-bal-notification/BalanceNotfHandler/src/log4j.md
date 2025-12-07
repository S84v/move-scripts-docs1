# Summary
The `log4j.properties` file configures Log4j for the **BalanceNotfHandler** component of the telecom move‑mediation system. It establishes a root logger at `INFO` level with two appenders: a `DailyRollingFileAppender` that writes to a file whose path is supplied via the `${logfile.name}` system/property variable, and a `ConsoleAppender` that streams logs to `System.out`. Both appenders use an identical pattern layout for timestamp, level, logger name, and message.

# Key Components
- **log4j.rootLogger** – Sets global logging level (`INFO`) and attaches the `File` and `stdout` appenders.  
- **log4j.appender.File** – `org.apache.log4j.DailyRollingFileAppender` writing to `${logfile.name}`; rotates daily.  
- **log4j.appender.File.layout** – `org.apache.log4j.PatternLayout` with pattern `[%‑5p] %d{ISO8601} %c - %m%n`.  
- **log4j.appender.stdout** – `org.apache.log4j.ConsoleAppender` targeting `System.out`.  
- **log4j.appender.stdout.layout** – Same `PatternLayout` as the file appender.

# Data Flow
- **Inputs**: System property or environment variable `logfile.name` (e.g., passed via JVM `-Dlogfile.name=/var/log/balance_notf.log`).  
- **Outputs**: Log records written to the daily‑rolled file and to the console (STDOUT).  
- **Side Effects**: File creation/rotation on the host filesystem; console output visible in batch job logs or container stdout.  
- **External Services**: None; purely internal to the Java process.

# Integrations
- Consumed by the Java runtime of **BalanceNotfHandler** during classpath initialization.  
- Works in conjunction with other configuration files (e.g., `MOVEDAO.properties`, `MNAAS_ShellScript.properties`) that drive the batch job; logging captures events from DAO operations, shell script wrappers, and Kafka/Kafka‑consumer interactions.  
- Log files may be harvested by external log aggregation tools (e.g., Splunk, ELK) via the file path defined by `${logfile.name}`.

# Operational Risks
- **Missing `logfile.name`** → Log4j defaults to a relative file path; may cause permission errors or fill the working directory. *Mitigation*: Enforce JVM `-Dlogfile.name` in launch scripts; validate at startup.  
- **Unbounded file growth** → DailyRollingFileAppender does not purge old files. *Mitigation*: Implement external log‑rotation or retention policy (cron, logrotate).  
- **Console log leakage** → In production containers, STDOUT may be captured by orchestration logs, increasing storage usage. *Mitigation*: Adjust log level or redirect console output in production profiles.

# Usage
```bash
# Example launch (Linux)
export LOGFILE_NAME=/opt/balance_notf/logs/balance_notf_$(date +%Y%m%d).log
java -Dlogfile.name=$LOGFILE_NAME -cp lib/*:conf/ BalanceNotfHandlerMain \
     -Dlog4j.configuration=file:/path/to/log4j.properties
```
*Debug*: Set `log4j.rootLogger=DEBUG, File, stdout` temporarily to increase verbosity.

# configuration
- **Environment Variable / JVM Property**: `logfile.name` – absolute path for the rolling log file.  
- **Referenced Config File**: `log4j.properties` (this file) – must be on the classpath or specified via `-Dlog4j.configuration`.  
- **Related Files**: `MOVEDAO.properties`, `MNAAS_ShellScript.properties` (provide DB, Kafka, email settings used by the same component).

# Improvements
1. Replace `DailyRollingFileAppender` with `RollingFileAppender` + `TimeBasedRollingPolicy` (Log4j 1.2.17) or migrate to Log4j2 for better rollover and compression.  
2. Add a `log4j.appender.File.MaxBackupIndex` or external retention script to enforce log retention limits.