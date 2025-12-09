# Summary
`ReportExporterMain` is the entry‑point for the TCL custom report export job. It parses three command‑line arguments (process name, report date, TCL SECS ID), logs them, and delegates to `ReportWriter.exportReport` to generate and persist the report. Exceptions are caught and printed.

# Key Components
- **ReportExporterMain.main(String[] args)**
  - Parses `args[0]` → `processName`
  - Parses `args[1]` → `reportDate`
  - Parses `args[2]` → `tclSecsId`
  - Logs the received parameters.
  - Instantiates `ReportWriter` and calls `exportReport(processName, reportDate, tclSecsId)`.
  - Handles `SecurityException`, `IOException`, `ParseException`.

- **static fields**
  - `processName`, `reportDate`, `tclSecsId` – shared state for potential downstream static access.
  - `logger` – Java Util Logging instance for this class.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|-------------|----------------------|
| CLI | `args[0..2]` (process name, date, SECS ID) | Assigned to static fields; logged | None |
| Export | Parameters passed to `ReportWriter.exportReport` | Business logic (not shown) creates report files, possibly writes to HDFS/S3, updates DB, sends notifications | Generated report artifacts, possible DB writes, logs, exceptions |

External services are invoked indirectly via `ReportWriter` (e.g., Hive, Spark, file system, mail).

# Integrations
- **ReportWriter** (same package) – core report generation component; expected to use HiveConnection, GetReportHeader, etc.
- **Logging** – Java Util Logging; may be configured by external logging framework (log4j bridge, etc.).
- **Batch orchestration** – Typically launched by a scheduler (e.g., Oozie, Airflow) that supplies the three arguments.

# Operational Risks
- **Argument validation missing** – malformed or missing args cause `ArrayIndexOutOfBoundsException`. *Mitigation*: validate `args.length` and format before use.
- **Static mutable state** – fields are public static; concurrent executions could interfere. *Mitigation*: remove static fields or make them thread‑local.
- **Broad exception handling** – only prints stack trace; job may appear successful to orchestrator. *Mitigation*: propagate exception or exit with non‑zero status.
- **Logging not configurable** – default JUL may not integrate with centralized logging. *Mitigation*: externalize logger configuration.

# Usage
```bash
# From project root (after mvn package)
java -cp tclcustomreport-0.0.1-SNAPSHOT.jar com.tcl.custom.report.ReportExporterMain \
    DailyRevenueExport 2025-12-07 SEC12345
```
For debugging, run from IDE with program arguments set accordingly.

# Configuration
- No environment variables or external config files are referenced directly in this class.
- Logging configuration may be supplied via `logging.properties` on the classpath.
- `ReportWriter` may read additional configuration (e.g., Hive JDBC URL, output paths) defined elsewhere in the module.

# Improvements
1. **Add robust argument validation** – check count, format (date parsing), and non‑null values; exit with clear error code if invalid.
2. **Refactor static fields** – make them local variables or encapsulate in a context object; remove public exposure to avoid side effects in concurrent runs.