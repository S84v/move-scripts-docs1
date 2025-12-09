# Summary
`log4j.properties` configures Log4j for the `SIMInvLoader` component of the Move mediation pipeline. It defines a root logger that writes INFO‑level (and above) messages to a daily‑rolling file and to the console (stdout), establishing a single source of logging behavior for the Java processes invoked by the `SIMInvLoader` job.

# Key Components
- **log4j.rootLogger** – Sets the root logging level to INFO and attaches the `File` and `stdout` appenders.  
- **log4j.appender.File** – `org.apache.log4j.DailyRollingFileAppender` that creates a new log file each day; file path is supplied via `${logfile.name}` system property.  
- **log4j.appender.File.layout** – `PatternLayout` with conversion pattern `[%-5p] %d{ISO8601} %c - %m%n` (level, timestamp, logger name, message).  
- **log4j.appender.stdout** – `org.apache.log4j.ConsoleAppender` targeting `System.out`.  
- **log4j.appender.stdout.layout** – Same `PatternLayout` as the file appender.

# Data Flow
- **Inputs**: Java system property `logfile.name` (provided by the invoking script or environment).  
- **Outputs**: Log entries written to the daily‑rolling file and to the console stream captured by the job’s stdout.  
- **Side Effects**: Creation of a new log file each day; potential disk I/O load.  
- **External Services**: None; pure in‑process logging.

# Integrations
- Sourced by `SIMInvLoader` Java classes (e.g., `com.move.siminventory.loader.Main`).  
- Invoked indirectly by the shell driver `SIMInvLoader.sh`, which sets `logfile.name` before launching the Java process.  
- Consumes the global `MNAAS_CommonProperties.properties` for environment paths, ensuring consistent log locations across jobs.

# Operational Risks
- **Missing `logfile.name`** → logging fallback to default location or failure to create file. *Mitigation*: Validate property in driver script; provide default path.  
- **Disk saturation** due to unbounded log growth. *Mitigation*: Enforce retention policy via OS logrotate or limit file size.  
- **Incorrect log level** (e.g., DEBUG in production) causing performance impact. *Mitigation*: Lock root level to INFO in production config; use separate dev config for verbose logging.

# Usage
```bash
# Set log file location
export logfile.name=/var/log/move/siminv_loader_$(date +%Y-%m-%d).log

# Run the loader (driver script sources this properties file)
./SIMInvLoader.sh
```
To debug logging configuration, run the Java class with `-Dlog4j.debug=true` to see Log4j initialization details.

# Configuration
- **Environment Variable**: `logfile.name` – absolute path for the daily rolling log file.  
- **Referenced Config Files**: None within this file; relies on external scripts to supply required system properties.

# Improvements
1. Add a size‑based rolling policy (`RollingFileAppender`) with a max file size to complement daily rolling and prevent oversized daily logs.  
2. Include a `log4j.appender.File.MaxBackupIndex` or integrate with OS `logrotate` to enforce retention and automatic cleanup of old log files.