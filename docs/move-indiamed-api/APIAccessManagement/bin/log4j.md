**File:** `move-indiamed-api\APIAccessManagement\bin\log4j.properties`  

---

## 1. High‑Level Summary
This `log4j.properties` file defines the logging behaviour for the **API Access Management** Java component of the Move‑Indiamed data‑move platform. It configures Log4j 1.x to emit INFO‑level messages (and above) to two destinations: a daily‑rolling file whose name is supplied at runtime via the `${logfile.name}` placeholder, and the console (STDOUT). The format includes log level, ISO‑8601 timestamp, logger name, and the message.

---

## 2. Key Configuration Items & Their Responsibilities  

| Item | Responsibility |
|------|-----------------|
| `log4j.rootLogger = INFO, File, stdout` | Sets the root logger level to **INFO** and attaches the two appenders defined below. |
| `log4j.appender.File` | **DailyRollingFileAppender** – writes logs to a file that rolls over each day. |
| `log4j.appender.File.File = ${logfile.name}` | File path is injected at runtime (system property or environment variable). |
| `log4j.appender.File.layout` | Uses `PatternLayout` with pattern `[%‑5p] %d{ISO8601} %c - %m%n`. |
| `log4j.appender.stdout` | **ConsoleAppender** – streams the same formatted log lines to `System.out`. |
| `log4j.appender.stdout.Target = System.out` | Explicitly routes console output to STDOUT (useful when the process is launched by a scheduler or container). |

*No Java classes or functions are defined in this file; it is consumed by any Java class that initializes Log4j (e.g., `org.apache.log4j.PropertyConfigurator.configure(...)`).*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - System property or environment variable `logfile.name` that resolves to an absolute or relative file path.<br>- Optional JVM system property `log4j.configuration` pointing to this file (usually set by the launch script). |
| **Outputs** | - Log file (daily‑rolled) written to the location supplied by `${logfile.name}`.<br>- Console output on STDOUT (captured by the process supervisor, e.g., `systemd`, `cron`, or a container log driver). |
| **Side‑effects** | - Disk I/O for each log entry; if the file path points to a full filesystem, the Java process may block or abort.<br>- Log rotation creates a new file each day; old files are retained indefinitely unless a cleanup job exists. |
| **Assumptions** | - The host JVM runs with Log4j 1.x on the classpath.<br>- The directory containing `${logfile.name}` exists and is writable by the process user.<br>- No other Log4j configuration (e.g., `log4j.xml`) overrides this file. |

---

## 4. Interaction with Other Scripts / Components  

| Component | Connection Point |
|-----------|------------------|
| **Java binaries** in `move-indiamed-api\APIAccessManagement\bin\` (e.g., `APIAccessDAO.jar`) | At startup they invoke `PropertyConfigurator.configure("log4j.properties")` or rely on the default Log4j lookup mechanism. |
| **Launch wrapper scripts** (bash, PowerShell, or scheduler jobs) | Must export `logfile.name` (e.g., `export logfile.name=/var/log/move/api-access-%d{yyyy-MM-dd}.log`) before invoking the Java main class. |
| **Monitoring / Alerting** (e.g., Splunk, ELK) | Consumes the STDOUT stream or the rolled log files for metric extraction. |
| **Other DDL / Hive scripts** (e.g., `move_sim_inventory_status_monthly.hql`) | No direct dependency, but they share the same runtime environment; consistent logging helps trace end‑to‑end data‑move jobs. |
| **Configuration property file** `APIAccessDAO.properties` | May contain a property that points to the log file name; the Java code typically reads that property and sets `logfile.name` accordingly. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or unreadable `${logfile.name}`** | No log file; possible loss of audit trail; Java may fall back to console only. | Validate the variable in the launch script; fail fast if the directory is not writable. |
| **Unbounded log file growth** (no retention policy) | Disk exhaustion → job failures. | Implement a cron/retention job to purge files older than N days, or switch to `RollingFileAppender` with size limits. |
| **Log4j 1.x vulnerability exposure** | Potential remote code execution if malicious log messages are processed. | Upgrade to Log4j 2.x or apply the official security patches; restrict logging of untrusted data. |
| **Incorrect log level (INFO) for production debugging** | Too much noise or insufficient detail. | Allow overriding via a system property (e.g., `-Dlog4j.rootLogger=DEBUG,File,stdout`) in non‑prod environments. |
| **Concurrent writes from multiple JVM instances to the same file** | Log interleaving, file lock contention. | Ensure each instance uses a unique file name (e.g., include PID or hostname). |

---

## 6. Running / Debugging the Component  

1. **Set the log file name** (example for a Linux cron job):  
   ```bash
   export logfile.name=/var/log/move/api-access-$(date +%Y-%m-%d).log
   java -Dlog4j.configuration=file:/opt/move-indiamed-api/APIAccessManagement/bin/log4j.properties \
        -cp /opt/move-indiamed-api/APIAccessManagement/bin/* com.mycompany.api.AccessManager
   ```
2. **Verify Log4j initialization** – the first line in the log file should be something like:  
   ```
   [INFO] 2025-12-04T10:15:23,456 org.apache.log4j.Logger - Root logger initialized
   ```
3. **Increase verbosity for troubleshooting** – add `-Dlog4j.rootLogger=DEBUG,File,stdout` to the JVM args.  
4. **Check file rotation** – after midnight, a new file should appear; the previous day's file remains unchanged.  
5. **Inspect console output** – when running interactively, the same messages appear on STDOUT, useful for immediate feedback.  

If logs are missing:
- Confirm `${logfile.name}` resolves correctly (`echo $logfile.name`).  
- Ensure the directory exists (`ls -ld $(dirname $logfile.name)`).  
- Look for Log4j warnings on STDERR (e.g., “Failed to locate appender”).  

---

## 7. External Configurations / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `${logfile.name}` | Path (including filename) for the daily‑rolling log file. Supplied via JVM system property (`-Dlogfile.name=…`) or OS environment variable. |
| `log4j.configuration` (optional) | Explicitly tells Log4j where to find this properties file; otherwise Log4j searches the classpath. |
| `APIAccessDAO.properties` | May contain a property that the Java code reads and then sets `logfile.name`. Verify the key name (e.g., `log.file.path`). |

---

## 8. Suggested TODO / Improvements  

1. **Migrate to Log4j 2.x** – provides better performance, built‑in security patches, and a more flexible configuration format (XML/JSON/YAML).  
2. **Add a retention/cleanup policy** – either switch to `RollingFileAppender` with a max backup index or schedule a daily purge script (`find /var/log/move -name 'api-access-*.log' -mtime +30 -delete`).  

--- 

*End of documentation.*