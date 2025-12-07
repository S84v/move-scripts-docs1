**File:** `move-indiamed-api\APIAccessManagementMigration\bin\log4j.properties`  

---

### 1. Purpose (one‑paragraph summary)  
This `log4j.properties` file defines the logging configuration for all Java components that belong to the **APIAccessManagementMigration** package (e.g., DAO utilities, transformation jobs, and any batch processes that move API‑access data between legacy and target systems). It sets the root logger to `INFO` level, directs log events to a daily‑rolling file appender whose filename is supplied at runtime via the `${logfile.name}` placeholder, and also mirrors the same output to the console (STDOUT). The configuration is loaded by the JVM at start‑up through the standard Log4j mechanism (`-Dlog4j.configuration=file:log4j.properties`).

---

### 2. Important Settings & Their Responsibilities  

| Setting | Responsibility |
|---------|-----------------|
| `log4j.rootLogger = INFO, File, stdout` | Sets default log level to **INFO** and attaches two appenders: a file appender (`File`) and a console appender (`stdout`). |
| `log4j.appender.File=org.apache.log4j.DailyRollingFileAppender` | Creates a file appender that rolls over each day, preventing a single massive log file. |
| `log4j.appender.File.File=${logfile.name}` | The actual log file path is injected at runtime (environment variable or JVM system property). |
| `log4j.appender.File.layout=org.apache.log4j.PatternLayout` | Formats each log entry. |
| `log4j.appender.File.layout.ConversionPattern=[%-5p] %d{ISO8601} %c - %m%n` | Example output: `[INFO ] 2025-12-04T14:23:01,123 com.tcl.api.dao - DAO operation completed` |
| `log4j.appender.stdout=org.apache.log4j.ConsoleAppender` | Sends the same log events to the console (useful for interactive debugging or when running under a scheduler that captures STDOUT). |
| `log4j.appender.stdout.Target=System.out` | Explicitly binds the console appender to STDOUT. |
| `log4j.appender.stdout.layout=org.apache.log4j.PatternLayout` | Same pattern as file appender for consistency. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - JVM system property or environment variable `logfile.name` (full path to the log file).<br>- Optional system property `log4j.debug` (if enabled, Log4j will emit its own internal diagnostics). |
| **Outputs** | - Daily‑rolled log file written to `${logfile.name}`.<br>- Log lines echoed to STDOUT (captured by the process supervisor, e.g., `systemd`, `cron`, or a batch scheduler). |
| **Side‑effects** | - Creation of log files on the host filesystem; may affect disk usage.<br>- Console output may be mixed with other process output if not isolated. |
| **Assumptions** | - The directory referenced by `${logfile.name}` exists and is writable by the Java process.<br>- The host OS supports file rotation via `DailyRollingFileAppender` (standard on Linux/Unix/Windows).<br>- No other Log4j configuration (e.g., `log4j.xml`) is loaded that would override these settings. |

---

### 4. Connection to Other Scripts & Components  

| Component | How it uses this file |
|-----------|-----------------------|
| **Java migration jobs** (`*.jar` under `APIAccessManagementMigration/bin`) | Launched with `-Dlog4j.configuration=file:log4j.properties`. The jobs import model classes (`ProductData`, `UsageDetail`, etc.) and DAO classes that rely on Log4j for audit and error reporting. |
| **Batch scheduler / CI pipeline** | Sets `logfile.name` (e.g., `/var/log/tcl/api-migration-${date}.log`) before invoking the Java command. |
| **Monitoring/Alerting tools** | May tail the generated log file to detect `ERROR` or `WARN` patterns. |
| **APIAccessDAO.properties** (sibling config) | Provides DB connection details; the DAO implementation logs connection attempts using the same Log4j configuration. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Missing or unreadable `${logfile.name}`** – Log4j will fail to open the file and may fall back to console only, losing persistent audit trails. | Validate the environment variable before job start; fail fast with a clear message if the path is not writable. |
| **Uncontrolled log growth** – Daily files can accumulate quickly, especially if `DEBUG` level is accidentally enabled. | Implement a log‑retention script (e.g., `find /var/log/tcl -name "api-migration-*.log" -mtime +30 -delete`). |
| **Permission issues** – The Java process may run under a limited user that cannot write to the target directory. | Ensure the execution user has `rw` rights on the log directory; document required OS permissions. |
| **Incorrect log level** – Production should not run at `DEBUG` due to performance and PII exposure. | Enforce `INFO` as default; optionally externalize the level to a property (`log4j.rootLogger=${log.level}, File, stdout`). |
| **Log file name collision** – If multiple concurrent jobs use the same `${logfile.name}`, logs intermix. | Include a unique identifier (job ID, timestamp) in the filename. |

---

### 6. Example: Running / Debugging the Migration Job  

```bash
# 1. Export the log file location (ensure the directory exists)
export LOGFILE_NAME="/var/log/tcl/api-migration-$(date +%Y%m%d-%H%M%S).log"

# 2. Launch the Java migration jar with explicit Log4j config
java -Dlog4j.configuration=file:log4j.properties \
     -Dlogfile.name="${LOGFILE_NAME}" \
     -jar move-indiamed-api/APIAccessManagementMigration/bin/api-migration.jar \
     --input /data/incoming/usage.csv \
     --output /data/outgoing/usage_processed.json

# 3. To debug Log4j configuration issues, enable internal diagnostics:
export LOG4J_DEBUG=true
java -Dlog4j.debug=true -Dlog4j.configuration=file:log4j.properties ...

# 4. Verify the log file was created and contains expected entries:
tail -f "${LOGFILE_NAME}"
```

*If the job aborts with a `FileNotFoundException` related to the log file, check that `LOGFILE_NAME` points to a writable directory and that the variable is correctly passed (`-Dlogfile.name`).*

---

### 7. External Config / Environment Variables Referenced  

| Variable | Source | Usage |
|----------|--------|-------|
| `logfile.name` | Set by the invoking shell script, CI pipeline, or batch scheduler (often via `export LOGFILE_NAME=...` then `-Dlogfile.name=${LOGFILE_NAME}`) | Determines the absolute path of the daily‑rolling log file. |
| `log4j.debug` (optional) | JVM system property (`-Dlog4j.debug=true`) | Enables Log4j internal debugging output (useful only for troubleshooting). |

No other external files are directly referenced by this properties file.

---

### 8. Suggested TODO / Improvements  

1. **Externalize the log level** – Replace the hard‑coded `INFO` with a placeholder (e.g., `${log.level}`) so operations can raise the level to `DEBUG` for a specific run without editing the file.  
2. **Add a size‑based rollover** – Replace `DailyRollingFileAppender` with `RollingFileAppender` (or Log4j 2’s `TimeBasedRollingPolicy`) to limit individual file size and avoid extremely large daily logs on high‑throughput days.  

---