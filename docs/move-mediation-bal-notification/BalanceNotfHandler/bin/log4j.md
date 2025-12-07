# Summary
The `log4j.properties` file configures Log4j for the **BalanceNotfHandler** component. It defines a root logger at `INFO` level with two appenders: a `DailyRollingFileAppender` that writes to a file whose name is supplied via the `${logfile.name}` property, and a `ConsoleAppender` that streams logs to `System.out`. The layout pattern standardises timestamp, level, logger name, and message formatting for both destinations.

# Key Components
- **log4j.rootLogger** – Sets the root logging level (`INFO`) and attaches the `File` and `stdout` appenders.  
- **log4j.appender.File** – `org.apache.log4j.DailyRollingFileAppender` that creates a new log file each day.  
  - `log4j.appender.File.File` – Path resolved from `${logfile.name}` (system property).  
  - `log4j.appender.File.layout` – `org.apache.log4j.PatternLayout`.  
  - `log4j.appender.File.layout.ConversionPattern` – `[%-5p] %d{ISO8601} %c - %m%n`.  
- **log4j.appender.stdout** – `org.apache.log4j.ConsoleAppender` targeting `System.out`.  
  - `log4j.appender.stdout.layout` – Same `PatternLayout` as the file appender.  
  - `log4j.appender.stdout.layout.ConversionPattern` – Identical pattern to the file appender.

# Data Flow
| Element | Description |
|---------|-------------|
| **Input** | System property `logfile.name` (e.g., `-Dlogfile.name=/var/log/balance_notf.log`). |
| **Processing** | Log4j initialises on JVM start, reads this properties file from the classpath, creates the two appenders, and applies the pattern layout. |
| **Output** | Log entries written to the daily‑rolled file and echoed to the console (STDOUT). |
| **Side Effects** | Creation of log files, daily rollover, potential file descriptor usage, and console output that may be captured by container logs. |
| **External Services** | File system (write permissions, disk space). No network or DB interactions. |

# Integrations
- **Java classes** in `BalanceNotfHandler` (e.g., `BalanceNotfHandlerMain`, DAO/processor classes) obtain a `org.apache.log4j.Logger` instance via `Logger.getLogger(ClassName.class)`. The logger inherits the root configuration defined here.
- **Build/Deployment** scripts must place `log4j.properties` on the runtime classpath (e.g., under `src/main/resources` or `bin`).  
- **Container/VM** environments may inject `logfile.name` via JVM arguments, environment variables, or orchestration templates.

# Operational Risks
1. **Missing `logfile.name` property** – Log4j falls back to an empty filename, causing `FileNotFoundException`.  
   *Mitigation*: Enforce the property at startup; fail fast if undefined.
2. **Insufficient file‑system permissions or disk space** – Logging may be silently dropped or cause application errors.  
   *Mitigation*: Monitor `/var/log` usage; run the process under a user with write access to the target directory.
3. **Excessive log volume** – `INFO` level may generate large daily files, impacting storage and I/O.  
   *Mitigation*: Adjust log level via property override for production; implement log rotation based on size if needed.
4. **Configuration drift** – Multiple copies of `log4j.properties` across environments can lead to inconsistent logging.  
   *Mitigation*: Centralise the file in version‑controlled configuration repository; validate checksum during deployment.

# Usage
```bash
# Example JVM launch
java -Dlogfile.name=/var/log/balance_notf.log \
     -cp "lib/*:conf/" com.tcl.move.balance.BalanceNotfHandlerMain \
     -config /etc/balance_notf/conf.properties
```
- Ensure `log4j.properties` is on the classpath (`conf/` in the example).  
- Verify that the log file path is writable before starting.  
- To debug logging configuration, add `-Dlog4j.debug=true` to the JVM arguments.

# Configuration
- **System Property**: `logfile.name` – absolute path for the daily rolling log file.  
- **File Location**: `log4j.properties` must reside in a directory included in the Java classpath (commonly `src/main/resources` or a `conf/` directory packaged with the jar).  
- **Optional Overrides**: `log4j.rootLogger` can be overridden at runtime with `-Dlog4j.rootLogger=DEBUG,File,stdout` if a higher verbosity is required.

# Improvements
1. **Migrate to Log4j 2** – Gain asynchronous logging, better performance, and native support for size‑based rolling policies.  
2. **Externalise Log Level** – Introduce a placeholder `${log.level}` with a default (`INFO`) and allow runtime override via environment variable or system property to avoid code redeployment for level changes.