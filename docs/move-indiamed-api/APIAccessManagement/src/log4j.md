**File:** `move-indiamed-api\APIAccessManagement\src\log4j.properties`

---

## 1. High‑Level Summary
This `log4j.properties` file defines the logging configuration for the **APIAccessManagement** Java component of the *move‑indiamed‑api* data‑move platform. It sets the root logger to `INFO` level and routes log events to two appenders: a daily‑rolling file appender (path supplied by the `${logfile.name}` variable) and a console appender that writes to `System.out`. The format includes log level, ISO‑8601 timestamp, logger name, and the message.

---

## 2. Key Configuration Elements  

| Element | Responsibility |
|---------|-----------------|
| `log4j.rootLogger = INFO, File, stdout` | Sets default log level to **INFO** and attaches the **File** and **stdout** appenders to every logger that does not have a more specific configuration. |
| `log4j.appender.File` (DailyRollingFileAppender) | Writes logs to a file whose name is supplied at runtime via `${logfile.name}`. The file rolls over daily, creating a new file for each day. |
| `log4j.appender.File.layout` (PatternLayout) | Formats each log line as `[LEVEL] yyyy‑MM‑dd'T'HH:mm:ss,SSS logger - message`. |
| `log4j.appender.stdout` (ConsoleAppender) | Mirrors the same log output to the standard output stream, useful for interactive debugging or when the process runs under a container that captures stdout. |
| `log4j.appender.stdout.layout` (PatternLayout) | Identical format to the file appender, ensuring consistency between file and console logs. |

*No explicit logger‑specific entries are present; all Java classes under `APIAccessManagement` inherit this root configuration unless overridden elsewhere.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - System property or environment variable `logfile.name` that resolves to an absolute file path (e.g., `/var/log/apiaccess/apiaccess.log`). |
| **Outputs** | - Log file(s) created under the path supplied by `logfile.name`. <br>- Console output to `stdout` (captured by the process manager, e.g., systemd, Docker logs, or YARN). |
| **Side Effects** | - Disk I/O for each log event. <br>- Potential file‑system growth if log rotation or retention is not managed. |
| **Assumptions** | - The directory containing `${logfile.name}` exists and is writable by the Java process. <br>- The Java runtime includes Log4j 1.x on the classpath (the component still uses the legacy Log4j API). <br>- No other `log4j.properties` files are loaded later in the classpath that would override these settings. |

---

## 4. Integration with Other Scripts & Components  

| Component | Connection Point |
|-----------|------------------|
| **Java source code** (`APIAccessManagement` package) | The component loads this file automatically via `LogManager.getLogger(...)` when the JVM starts, provided the file is on the classpath (`src/main/resources` or bundled in the JAR). |
| **APIAccessDAO.properties** (sibling config) | Not directly related, but both files are packaged together; the DAO properties supply DB connection details, while this file supplies runtime logging. |
| **Deployment scripts / CI pipelines** | Build scripts (Maven/Gradle) copy `src/log4j.properties` into the final artifact. Deployment automation may inject the `logfile.name` value via environment variables or JVM `-Dlogfile.name=...`. |
| **Monitoring / Log aggregation** | Console output is typically captured by the container orchestrator (e.g., Kubernetes) and forwarded to a log‑aggregation system (ELK, Splunk). The rolling file may be harvested by a log‑shipping daemon (Filebeat, Fluentd). |
| **Other DDL / Hive scripts** | No functional coupling; the logging configuration is independent of the Hive DDL files listed in the history, but all components share the same operational environment (e.g., same host, same log retention policies). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or unreadable `${logfile.name}`** | Application fails to start or silently discards file logs. | Validate the variable at startup; fail fast with a clear error message. Provide a default fallback path in the deployment script. |
| **Unbounded log file growth** | Disk exhaustion leading to service outage. | Enforce a retention policy via external log‑rotation (e.g., `logrotate`) or switch to `RollingFileAppender` with size‑based rollover. |
| **Log4j 1.x security vulnerabilities** (e.g., CVE‑2021‑4104) | Potential remote code execution if untrusted data is logged. | Upgrade to Log4j 2.x or apply the official security patches; restrict logging of user‑controlled data. |
| **Incorrect log level in production** (INFO may be too verbose) | Performance impact, noisy logs. | Allow log level override via `-Dlog4j.rootLogger=WARN,File,stdout` in the launch script. |
| **Concurrent writes to the same log file from multiple JVMs** | Log interleaving, file lock contention. | Use separate log files per instance (include hostname/PID in `${logfile.name}`) or switch to a centralized logging service. |

---

## 6. Example: Running / Debugging the Component  

1. **Set the log file path** (environment variable or JVM property).  
   ```bash
   export LOGFILE_NAME=/var/log/apiaccess/apiaccess.log
   # or pass as a JVM arg
   java -Dlogfile.name=$LOGFILE_NAME -jar apiaccess-management.jar
   ```

2. **Start the service** (typical script).  
   ```bash
   #!/bin/bash
   LOGFILE=/var/log/apiaccess/$(date +%Y-%m-%d)_apiaccess.log
   java -Dlogfile.name=$LOGFILE -cp lib/*:conf/ move.indiamed.api.APIOperationMain
   ```

3. **Verify logging**:  
   - Tail the console output: `tail -f /var/log/apiaccess/$(date +%Y-%m-%d)_apiaccess.log`  
   - Check that daily rollover creates a new file after midnight.  

4. **Debugging missing logs**:  
   - Confirm the property is resolved: `java -Dlogfile.name=$LOGFILE -verbose:class ...` and look for `log4j.properties` loading messages.  
   - Enable Log4j internal debugging by adding `-Dlog4j.debug=true` to the JVM args.  

---

## 7. External Config / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `${logfile.name}` | Fully qualified path for the daily‑rolling file appender. Must be supplied at runtime (env var, system property, or via a wrapper script). |
| `log4j.properties` (this file) | Loaded automatically from the classpath; any other `log4j.properties` placed earlier in the classpath will override these settings. |
| `APIAcessMgmt.properties` (sibling) | Provides business‑logic configuration (DB URLs, API keys). Not directly used by logging but often co‑deployed. |

---

## 8. Suggested TODO / Improvements  

1. **Migrate to Log4j 2.x** – eliminates known security issues, provides asynchronous appenders, and richer configuration (XML/JSON/YAML).  
2. **Add size‑based rollover and retention** – replace `DailyRollingFileAppender` with `RollingFileAppender` configured with `MaxFileSize` and `MaxBackupIndex` to bound disk usage.  

--- 

*End of documentation.*