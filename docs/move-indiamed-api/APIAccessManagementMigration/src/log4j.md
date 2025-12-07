**File:** `move-indiamed-api\APIAccessManagementMigration\src\log4j.properties`  

---

## 1. High‑Level Summary
This file provides the Log4j 1.x configuration for the *API Access Management Migration* job. It defines a root logger that writes INFO‑level (and above) messages to two appenders: a daily‑rolling file appender whose filename is supplied at runtime via the `${logfile.name}` variable, and a console appender that streams to `System.out`. All Java classes in the migration package that use Log4j will inherit this configuration automatically when the file is placed on the application classpath.

---

## 2. Key Configuration Elements  

| Element | Responsibility |
|---------|-----------------|
| `log4j.rootLogger = INFO, File, stdout` | Sets the default logging level to **INFO** and attaches the two appenders (`File` and `stdout`). |
| `log4j.appender.File` | Declares a **DailyRollingFileAppender** that creates a new log file each day. |
| `log4j.appender.File.File=${logfile.name}` | The **target log file path**; resolved from the system property `logfile.name` supplied at runtime. |
| `log4j.appender.File.layout` / `ConversionPattern` | Formats each log entry as `"[LEVEL] timestamp logger - message"`. |
| `log4j.appender.stdout` | Declares a **ConsoleAppender** that writes to `System.out`. |
| `log4j.appender.stdout.layout` / `ConversionPattern` | Same message format as the file appender. |

*No custom logger categories are defined here; any class‑level logger (`Logger.getLogger(<class>)`) will inherit the root configuration.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - System property `logfile.name` (e.g., `-Dlogfile.name=/var/log/indiamed/api-access-migration.log`). |
| **Outputs** | - Daily‑rolled log file on the filesystem (path defined by `logfile.name`).<br>- Console output to STDOUT (visible in the job’s execution console or container logs). |
| **Side Effects** | - Disk I/O for each log statement.<br>- Creation of new log files each day (potentially many files over time). |
| **Assumptions** | - The directory containing `${logfile.name}` exists and is writable by the process.<br>- Sufficient disk space for daily logs.<br>- Log4j 1.x library is on the classpath (the job uses the legacy API).<br>- No other Log4j configuration files with higher precedence are present in the classpath. |

---

## 4. Integration with Other Scripts & Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **Java source files** (`*.java` under `src/`) | All classes that obtain a Log4j `Logger` (e.g., `Logger logger = Logger.getLogger(MyClass.class)`) automatically use this configuration because the file is packaged in the same JAR/classpath. |
| **Job launcher scripts** (e.g., shell or Ant scripts that start the migration) | Must set the `logfile.name` system property before invoking the Java main class, e.g., `java -Dlogfile.name=$LOG_DIR/api-access-migration.log -cp ... com.company.migration.Main`. |
| **Properties files** (`APIAcessMgmt.properties`, `APIAccessDAO.properties`) | Not directly referenced, but they are part of the same job; their logs will also flow through this configuration. |
| **Monitoring/Alerting tools** (e.g., Splunk, ELK) | May ingest the generated log files; the consistent pattern makes parsing straightforward. |
| **CI/CD pipelines** | When the job is executed in a CI stage, the console appender provides immediate feedback; the file appender can be redirected to an artifact for post‑run analysis. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing `logfile.name` property** | Log4j will attempt to write to a literal `${logfile.name}` file, causing file‑not‑found errors or writing to an unintended location. | Enforce the property in the launch script; add a pre‑flight check that aborts if the variable is unset. |
| **Disk space exhaustion** | Logging stops, job may fail, or old logs may be overwritten unintentionally. | Implement a log‑retention policy (e.g., cron job to delete files older than X days) and monitor filesystem usage. |
| **Excessive log volume** (e.g., DEBUG statements accidentally enabled) | Performance degradation, larger storage consumption. | Keep the root level at INFO; use a separate logger configuration for debug builds if needed. |
| **File permission errors** | Job cannot create or write to the log file, leading to runtime exceptions. | Ensure the process user has write permission on the target directory; validate on start‑up. |
| **Log4j 1.x end‑of‑life** | No security patches; potential vulnerability exposure. | Plan migration to Log4j 2.x or another supported logging framework; encapsulate the config in a version‑controlled module. |

---

## 6. Running / Debugging the Job  

1. **Set the log file path** (example in Bash):  
   ```bash
   export LOG_DIR=/var/log/indiamed
   export LOG_FILE=${LOG_DIR}/api-access-migration.log
   ```
2. **Launch the Java job** (simplified):  
   ```bash
   java -Dlogfile.name=${LOG_FILE} \
        -cp target/api-access-migration.jar:lib/* \
        com.company.indiamed.migration.Main
   ```
3. **Verify logging**:  
   - Console: Look for lines prefixed with `[INFO]` in the terminal.  
   - File: `tail -f ${LOG_FILE}` should show the same messages, rolled over daily.  
4. **Debugging missing logs**:  
   - Confirm that `logfile.name` is set (`echo $logfile.name`).  
   - Check classpath for duplicate `log4j.properties` files that may override this one (`grep -R "log4j.rootLogger" $CLASSPATH`).  
   - Increase the root level temporarily (`log4j.rootLogger = DEBUG, File, stdout`) to see if lower‑level messages appear.  

---

## 7. External Config / Environment Variables  

| Variable | Purpose | Where it is used |
|----------|---------|------------------|
| `logfile.name` | Full path (including filename) of the daily‑rolling log file. | Referenced directly in `log4j.appender.File.File=${logfile.name}`. |
| (Implicit) `log4j.configuration` | If set, can override the default location of this file. | Not required for normal operation; useful for testing alternate configs. |

---

## 8. Suggested Improvements (TODO)

1. **Migrate to Log4j 2.x** – provides better performance, asynchronous appenders, and built‑in security fixes.  
2. **Externalize the log level** – replace the hard‑coded `INFO` with a system property (e.g., `${log.level:-INFO}`) so operators can increase verbosity without code changes.  

---