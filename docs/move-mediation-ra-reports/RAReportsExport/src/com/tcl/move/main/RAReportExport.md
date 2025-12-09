# Summary
`RAReportExport` is the entry‑point batch program that orchestrates data extraction from Hadoop‑based mediation tables, transforms the data into domain DTOs, delegates Excel export to `ExportService`, and optionally distributes the generated reports via `MailService` or `SCPService`. It is invoked by a cron/shell wrapper with command‑line arguments that specify the report type, date parameters, property file, and logging destination.

# Key Components
- **`main(String[] args)`** – parses CLI arguments, loads system properties, creates `RAReportExport` instance, invokes `exportData` and conditional `sendReports`.
- **`exportData(String repType, String dateMonthYear, String monthYear)`** – core workflow:
  - Moves previous reports (`Utils.moveFiles`).
  - Determines target dates (yesterday, first/last day of month) with fallback logic.
  - Calls `MediaitionFetchService` methods to retrieve DTO collections for the selected `repType`.
  - Logs record counts.
  - Calls corresponding `ExportService` method to generate Excel files.
  - Handles `DatabaseException` with alert mail.
- **`sendReports(String repType)`** – scans the extraction directory, builds a list of report files, and dispatches:
  - `MailService.sendReport` for email‑able types.
  - `SCPService.sendReport` for DPPU/MPPU types.
  - Sends alert mail on failure and wraps the error in `FileException`.
- **Utility methods**
  - `getStackTrace(Exception)` – converts stack trace to string for logging.
- **Member fields**
  - Service instances (`MediaitionFetchService`, `ExportService`, `MailService`, `SCPService`).
  - `Date dailyReportDate`, `Date monthlyReportDate`.
  - Static `Logger logger`.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|--------------------|
| 1 | CLI args (`dateMonthYear`, `monthYear`, `propertyFile`, `logFile`, `repType`, `sendRep`) | Load properties → set system properties | System properties for DB, paths, etc. |
| 2 | `repType` | `exportData` selects branch | DTO lists from `MediaitionFetchService` (DB/Hadoop reads) |
| 3 | DTO lists | `ExportService` creates Excel files in `extract_path` | Physical Excel files |
| 4 (optional) | `sendRep = Y` & `repType != ipv` | `sendReports` reads files from `extract_path` | Email via `MailService` **or** SCP transfer via `SCPService` |
| 5 | Errors (FileException, DatabaseException, generic) | Log + alert mail via `MailService.sendAlertMail` | Alert email, batch termination |

External services:
- **`MediaitionFetchService`** – data access layer (Hadoop/Hive/DB).
- **`ExportService`** – Excel generation (Apache POI or similar).
- **`MailService`** – SMTP email & alert.
- **`SCPService`** – remote file copy (SCP).
- **`Utils`** – file movement, date utilities.

# Integrations
- **Property file** (`MNAAS_ShellScript.properties`) supplies:
  - `extract_path`, `backup_path`, DB connection strings, mail server config, SCP credentials.
- **Logging** – Log4j configured via `logFile` system property.
- **Database/Hadoop** – accessed indirectly through `MediaitionFetchService`; exceptions wrapped as `DatabaseException`.
- **Email** – `MailService` uses system mail properties.
- **SCP** – `SCPService` uses system properties for remote host/user/key.

# Operational Risks
- **Missing/invalid property file** → batch aborts; mitigated by early validation and alert mail.
- **File movement failure** → old reports not archived; mitigated by alert mail and manual intervention.
- **Date parsing errors** → defaults to yesterday/last month; ensure input format compliance.
- **Out‑of‑disk in `extract_path`** → export fails; monitor disk usage and set alerts.
- **Uncaught runtime exceptions** → generic catch logs stack trace; consider more granular handling.
- **Email/SCP failures** → alerts sent; verify SMTP/SCP connectivity before schedule.

# Usage
```bash
java -cp <classpath> com.tcl.move.main.RAReportExport \
    <dateMonthYear> <monthYear> <propertyFile> <logFile> <repType> <sendRep>
```
Example (daily report for 2023‑02‑01, send email):
```bash
java -cp lib/*:conf/ RAReportExport \
    2023-02-01 def_mon /opt/ra/conf/MNAAS_ShellScript.properties /var/log/RAExport.log daily Y
```
For debugging, set `logfile.name` to a writable location and run with `repType` = `daily` and `sendRep` = `N`.

# Configuration
- **System properties (populated from property file)**
  - `extract_path` – directory where Excel reports are written.
  - `backup_path` – directory for archiving previous reports.
  - DB connection parameters (`db.url`, `db.user`, `db.password`).
  - Mail settings (`mail.smtp.host`, `mail.smtp.port`, etc.).
  - SCP settings (`scp.host`, `scp.user`, `scp.keyfile`).
- **Constants** (in `Constants`):
  - Date format strings (`YMD_HYPHEN_FORMAT`, `YM_HYPHEN_FORMAT`, etc.).

# Improvements
1. **Refactor exception handling** – replace generic `catch (Exception)` blocks with specific typed catches; propagate meaningful error codes to the scheduler.
2. **Externalize report type logic** – use a strategy pattern or enum mapping `repType` to service handlers to simplify `exportData` and `sendReports` and improve testability.