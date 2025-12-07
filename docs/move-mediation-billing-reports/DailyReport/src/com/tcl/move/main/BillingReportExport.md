# Summary
`BillingReportExport` is the entry‑point batch program that orchestrates extraction of mediation data from Hadoop/Impala, populates a month‑reference table, generates daily or multiple monthly billing reports (subscription, usage, partner‑specific, SMS, consolidated), writes them to Excel/CSV files via `ExcelExport`, and distributes the results via `MailService`. It also manages logging, property loading, and graceful shutdown of the Impala connection.

# Key Components
- **`main(String[] args)`** – Parses command‑line arguments, loads properties, determines target month, invokes processing workflow, handles top‑level exceptions.  
- **`getMonthYear(String monthType)`** – Computes current or previous month in `yyyy‑MM` format.  
- **`shutdown()`** – Closes the Impala JDBC connection.  
- **`insertMonth(String monthYear)`** – Persists the reporting month into a reference table; on failure sends alert mail.  
- **`exportData(...)`** – Dispatches to daily or selected monthly export methods based on flags; wraps calls in error handling and alert mailing.  
- **Daily export methods** (`exportDailyReport`) – Retrieves SIM counts, traffic, eSIM transactions, usage; delegates to `ExcelExport.exportDailyReport`.  
- **Monthly export methods** (`exportMonthlySubscriptionReport`, `exportMonthlyUsageReport`, `exportJLRUsageReport`, `exportEluxUsageReport`, `exportNext360UsageReport`, `exportSMSReport`, `exportUsageReport`, `exportCompleteReport`) – Each fetches specific DTO lists from `MediaitionFetchDAO` and calls the corresponding `ExcelExport` routine.  
- **`getStackTrace(Exception)`** – Utility to convert stack trace to string for logging/mail.  

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| CLI | args[0‑11] (monthYear, propertyFile, logFile, flags) | Parsed in `main` | Variables for workflow | None |
| Property Load | `propertyFile` (path) | `Properties.load` → System properties | System properties for downstream components | Logs error if file missing |
| DB Insert | `monthYear` | `MediaitionFetchDAO.insertMonth` | Row in month‑reference table | None |
| Data Retrieval | `monthYear` + flags | Multiple DAO methods (`getHOLActives`, `getUsage`, etc.) | Lists of `ExportRecord` / `MonthlyExportRecord` / `BYONRecord` | DB reads from Impala/Hadoop |
| Report Generation | DTO lists | `ExcelExport` methods | Excel/CSV files on filesystem (path derived inside `ExcelExport`) | File I/O |
| Notification | Generated file name | `MailService.sendReport` / `sendAlertMail` | Email with attachment or alert | SMTP outbound |
| Shutdown | – | `JDBCConnection.closeImpalaConnection` | Closed JDBC connection | Resource release |

# Integrations
- **`MediaitionFetchDAO`** – Data Access Object for Impala/Hadoop queries; supplies all DTO collections.  
- **`ExcelExport`** – Service that builds Excel/CSV workbooks from DTOs; returns generated file name.  
- **`MailService`** – Sends report files and alert emails via configured SMTP server.  
- **`JDBCConnection`** – Manages Impala JDBC lifecycle; used for explicit close in `shutdown`.  
- **Log4j** – Logging configured via external `logFile` property (`logfile.name`).  
- **System properties** – Populated from `MNAAS_ShellScript.properties`; consumed indirectly by DAO/Export services.

# Operational Risks
- **Missing/invalid command‑line arguments** – Leads to `ArrayIndexOutOfBoundsException`. *Mitigation*: Validate `args.length` before use.  
- **Property file not found or malformed** – Causes startup failure. *Mitigation*: Add fallback defaults and explicit error exit.  
- **DAO exceptions not recovered** – Batch aborts without partial results. *Mitigation*: Implement retry logic for transient DB errors.  
- **Large DTO lists may cause OOM** – All records are loaded into memory before export. *Mitigation*: Stream results or paginate DAO queries.  
- **Email send failure** – Alerts may be lost. *Mitigation*: Queue alerts to a persistent mail queue (e.g., JMS) and retry.  
- **Hard‑coded flag ordering** – Changes in script wrapper can mis‑map flags. *Mitigation*: Use named arguments or a config file.

# Usage
```bash
java -cp <classpath> com.tcl.move.main.BillingReportExport \
  2023-07 \
  /opt/move/conf/MNAAS_ShellScript.properties \
  /var/log/move/Export.log \
  M \
  Y N Y N N Y Y Y Y
```
- Replace `<classpath>` with compiled classes and dependencies.  
- For debugging, set `logfile.name` to a writable path and run with a JVM debugger attached to `BillingReportExport.main`.

# Configuration
- **Property file** (`MNAAS_ShellScript.properties`) – Loaded at runtime; each key/value is copied to `System` properties for DAO/Export services.  
- **Log4j** – `logfile.name` system property determines log file location.  
- **Environment variables** – None directly referenced; all configuration is via the property file and command‑line args.  
- **SMTP settings** – Expected to be defined in the property file for `MailService`.  

# Improvements
1. **Argument Validation & Named Parameters** – Replace positional `args[]` with a robust CLI parser (e.g., Apache Commons CLI) to enforce required arguments, provide help text, and avoid ordering errors.  
2. **Streaming Data Retrieval** – Refactor DAO methods to return `Iterator<ExportRecord>` or use Apache Spark/DataFrames to process records in chunks, reducing memory footprint for large monthly datasets.