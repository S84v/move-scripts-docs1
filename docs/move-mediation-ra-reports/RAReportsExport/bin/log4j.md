# Summary
The `log4j.properties` file configures Log4j for the RAReportsExport component of the MOVE‑RA‑Reports subsystem. It defines a root logger at INFO level with two appenders: a daily‑rolling file appender (`File`) that writes to the path supplied by the `${logfile.name}` variable, and a console appender (`stdout`). All log statements from the application are formatted with a standard pattern and routed to both destinations, enabling persistent audit trails and real‑time console visibility during batch execution.

# Key Components
- **`log4j.rootLogger`** – Sets the root logging level (`INFO`) and attaches the `File` and `stdout` appenders.  
- **`log4j.appender.File`** – `org.apache.log4j.DailyRollingFileAppender` that creates a new log file each day.  
  - **`log4j.appender.File.File`** – Destination file path resolved from `${logfile.name}`.  
  - **`log4j.appender.File.layout`** – `PatternLayout` with conversion pattern `[%-5p] %d{ISO8601} %c - %m%n`.  
- **`log4j.appender.stdout`** – `org.apache.log4j.ConsoleAppender` targeting `System.out`.  
  - **`log4j.appender.stdout.layout`** – Same `PatternLayout` as the file appender.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input** | Environment variable `${logfile.name}` | Absolute or relative path where daily log files are written. |
| **Output** | `File` appender | Rotating log file on the filesystem (one file per day). |
| **Output** | `stdout` appender | Console (STDOUT) stream, captured by batch job logs or terminal. |
| **Side Effects** | File creation, file rotation, console output | Enables post‑run log aggregation and real‑time monitoring. |
| **External Services** | None (pure logging). |

# Integrations
- **RAReportsExport Java classes** – All classes using `org.apache.log4j.Logger` inherit this configuration via Log4j’s automatic property loading on classpath.  
- **Batch orchestration scripts** – Shell or scheduler wrappers that set `${logfile.name}` before launching the Java process, ensuring logs are placed in the correct run‑specific directory.  
- **Monitoring/Alerting tools** – Log aggregation platforms (e.g., Splunk, ELK) ingest the generated files for operational dashboards.

# Operational Risks
- **Missing `${logfile.name}`** – Logger falls back to a default path or fails to create the file, causing loss of audit logs. *Mitigation*: Validate the variable at job start; provide a fallback default in the script.  
- **Disk saturation** – Daily log files can accumulate quickly under high volume, leading to out‑of‑space errors. *Mitigation*: Implement log retention/cleanup policies; monitor filesystem usage.  
- **Incorrect file permissions** – Process may lack write permission, causing startup failure. *Mitigation*: Ensure the execution user has appropriate rights on the target directory.  
- **Log level too verbose** – INFO may generate excessive data in high‑throughput runs. *Mitigation*: Adjust `log4j.rootLogger` to `WARN` or `ERROR` for production, or use per‑package overrides.

# Usage
```bash
# Example batch wrapper
export logfile.name=/var/log/ra_reports/ra_report_$(date +%Y%m%d).log
java -Dlog4j.configuration=file:/opt/move/RAReportsExport/bin/log4j.properties \
     -cp /opt/move/RAReportsExport/lib/* com.tcl.ra.RAReportsExportMain \
     --input /data/kc_input.csv \
     --output /data/kc_output.parquet
```
- Set `logfile.name` before launching.  
- Verify that the generated log file appears and rotates at midnight.

# Configuration
- **Environment Variable**: `logfile.name` – Full path for the daily log file.  
- **Property File**: `log4j.properties` – Must be on the Java classpath or referenced via `-Dlog4j.configuration`.  
- No additional external configuration files are referenced.

# Improvements
1. **Add per‑package log level overrides** – Enable fine‑grained control (e.g., `log4j.logger.com.tcl.parquet=DEBUG`) without raising the global level.  
2. **Introduce size‑based rollover** – Replace `DailyRollingFileAppender` with `RollingFileAppender` with `MaxFileSize` and `MaxBackupIndex` to bound disk usage when daily volume spikes.