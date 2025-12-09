# Summary
`ReportGenerator` is the entry‑point for the MOVE‑Mediation‑KLM nightly/monthly batch. It loads runtime properties, inserts the billing month (and optionally traffic data) into the database, retrieves summary and detailed KLM records, exports them to an Excel workbook, and sends alert e‑mails on failures. It also closes the Impala JDBC connection before exiting.

# Key Components
- **`main(String[] args)`** – parses command‑line arguments, initializes logging, loads properties, drives the batch workflow, and handles top‑level exceptions.  
- **`insertMonthAndTraffic(String monthYear, boolean loadTraffic)`** – delegates to `MediationFetchService.insertMonthAndTraffic` and sends an alert mail on `DatabaseException`.  
- **`loadKLMReports(String monthYear)`** – obtains summary (`getKLMSummary`) and zone‑wise detail (`getKLMReportData`) records, aggregates IN/OUT lists, and calls `ExcelExport.exportKLMReport`. Handles `DatabaseException` with alert mail.  
- **`getMonthYear(String monthType)`** – utility to compute the previous month (format `yyyy-MM`).  
- **`shutdown()`** – closes the shared Impala connection via `JDBCConnection.closeImpalaConnection`.  
- **`getStackTrace(Exception e)`** – converts an exception stack trace to a `String` for logging/e‑mail.  

# Data Flow
| Step | Input | Processing | Output / Side Effect |
|------|-------|------------|----------------------|
| 1 | Command‑line args (`monthYear`, `propertyFile`, `logFile`, `loadMonthTraffic`) | Load properties file → set System properties | System properties for downstream services |
| 2 | `monthYear`, `loadMonthTraffic` | `MediationFetchService.insertMonthAndTraffic` inserts rows into `month_billing` and optionally `traffic_details_billing` tables | DB writes |
| 3 | `monthYear` | `MediationFetchService.getKLMSummary` → `Map<Integer,KLMRecord>`; `MediationFetchService.getKLMReportData` per zone → `Map<String,List<KLMRecord>>` | In‑memory collections |
| 4 | Aggregated maps | `ExcelExport.exportKLMReport` writes an Excel file to a location defined in properties | Excel file (report) |
| 5 | Exceptions (`DatabaseException`, generic) | Log error, compose e‑mail body, invoke `MailService.sendAlertMail` | Alert e‑mail |
| 6 | End of batch | `JDBCConnection.closeImpalaConnection` | DB connection closed |

External services:
- **Impala/JDBC** – accessed via `JDBCConnection`.
- **Mail server** – used by `MailService`.
- **File system** – for property file, log file, and generated Excel report.

# Integrations
- **`MediationFetchService`** (service layer) – provides DB CRUD for month/traffic tables and KLM data retrieval.  
- **`ExcelExport`** – consumes aggregated KLM data and creates the final report workbook.  
- **`MailService`** – sends alert e‑mails on failures.  
- **`JDBCConnection`** – singleton connection manager for Impala; closed in `shutdown()`.  
- **`log4j`** – logging configured via the supplied log file path (`logfile.name` system property).  

# Operational Risks
- **Missing/invalid property file** → batch aborts; mitigated by early validation and exit code 1.  
- **DatabaseException not rolled back** → partial inserts may remain; ensure `MediationFetchService` uses transactional boundaries.  
- **Alert e‑mail failure** → error logged but batch continues; consider retry or dead‑letter queue.  
- **Hard‑coded property keys** → changes require code update; externalize via a schema‑validated config file.  
- **Memory pressure** when aggregating large KLM detail lists; monitor JVM heap and consider streaming export.  

# Usage
```bash
java -cp <classpath> com.tcl.move.main.ReportGenerator \
    2023-07 \
    /opt/move/config/MNAAS_ShellScript.properties \
    /var/log/move/Export.log \
    Y
```
- Replace `<classpath>` with compiled classes and required libraries (log4j, JDBC driver, Apache POI, etc.).  
- For debugging, set `log4j.debug=true` in the properties file and run with a debugger attached to `ReportGenerator.main`.

# Configuration
- **Properties file** (path supplied as arg[1]) – contains all system properties referenced by services (e.g., DB connection strings, mail server settings, export directory).  
- **System property `logfile.name`** – set at runtime to direct Log4j output.  
- **Implicit properties** expected by downstream services:  
  - `db.url`, `db.user`, `db.password` (Impala)  
  - `mail.smtp.host`, `mail.smtp.port`, `mail.from`, `mail.to`  
  - `export.dir` (directory for generated Excel file)  

# Improvements
1. **Replace generic `Exception` handling with specific catches** (e.g., `IOException`, `SQLException`) and propagate meaningful error codes to the scheduler.  
2. **Introduce a batch‑level transaction manager** to ensure `insertMonthAndTraffic` and subsequent report generation are atomic; on failure, roll back DB changes and clean up partial Excel files.