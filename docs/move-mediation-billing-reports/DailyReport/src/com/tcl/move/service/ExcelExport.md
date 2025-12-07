# Summary
`ExcelExport` provides a collection of methods that generate daily, monthly, and usage reports for the MOVE mediation billing system. It creates Excel (`.xlsx`) workbooks with multiple sheets or CSV files, populates them from supplied DTO lists, applies basic styling, auto‑sizes columns, and writes the files to a directory defined by the `excel_path` system property.

# Key Components
- **class `ExcelExport`**
  - `exportDailyReport(...)` – builds a 5‑sheet daily billing workbook (Actives, Tolling, IPvProbe, eSimHub, Traffic) and saves it as *Daily Billing Report*.
  - `exportMonthlySubscriptionReport(...)` – builds an 8‑sheet monthly subscription workbook (Actives, Tolling, Activations, Addon, IPvProbe, eSimHub, BYON, Trimble Tolling) and saves it as *Monthly Subscription Billing Report*.
  - `exportMonthlyUsageReport(...)` – builds a 5‑sheet usage workbook (Traffic, Tadig‑Wise, Olympic, Emirates, Global‑Gig) and saves it as *Monthly Usage Billing Report*.
  - `exportJLRUsageReport(...)`, `exportEluxUsageReport(...)`, `exportNext360UsageReport(...)` – write CSV files for specific customers (JLR, E‑LUX, MOVE Next360).
  - `exportSMSReport(...)` – creates an Excel sheet for SMS usage.
  - `exportMoveReport(...)` – writes a CSV containing detailed MOVE CDR aggregation.
  - `exportUsageReport(...)` – writes a generic usage CSV.
  - `autoSizeColumns(XSSFWorkbook)` – iterates each sheet’s first row to auto‑size columns.
  - `getStackTrace(Exception)` – utility to convert an exception stack trace to a string for logging.

# Data Flow
| Method | Input Parameters | Primary Output | Side Effects |
|--------|------------------|----------------|--------------|
| `exportDailyReport` | `monthYear`, lists of `ExportRecord` for each category | Excel file `Daily Billing Report - <date>.xlsx` | Writes file to `excel_path`; logs errors |
| `exportMonthlySubscriptionReport` | `monthYear`, multiple `ExportRecord`/`BYONRecord` lists | Excel file `Monthly Subscription Billing Report - <monthYear>.xlsx` | Writes file; logs errors |
| `exportMonthlyUsageReport` | `monthYear`, usage lists | Excel file `Monthly Usage Billing Report - <monthYear>.xlsx` | Writes file; logs errors |
| CSV exporters (`exportJLRUsageReport`, `exportEluxUsageReport`, `exportNext360UsageReport`, `exportMoveReport`, `exportUsageReport`) | `monthYear`, relevant `ExportRecord` list | CSV file with naming pattern `<type> - <monthYear>.csv` | Writes file; logs errors |
| `exportSMSReport` | `monthYear`, list of SMS `ExportRecord` | Excel file `Monthly SMS Report - <monthYear>.xlsx` | Writes file; logs errors |

All methods rely on **DTOs** (`ExportRecord`, `MonthlyExportRecord`, `BYONRecord`) that already contain the data to be written. No direct DB, queue, or external service calls are performed within this class.

# Integrations
- **`ExportUtils`** – static helper used to fill common sheet structures (Actives, Tolling, IPvProbe, eSimHub, Usage, Addon). `ExcelExport` delegates row creation for those sheets to this utility.
- **System Property `excel_path`** – external configuration that determines the output directory; set by the surrounding application or container.
- **Logging** – uses Log4j (`org.apache.log4j.Logger`) for error reporting.
- **Apache POI** – library for Excel workbook creation and manipulation.
- **File I/O** – standard Java `FileOutputStream` and `FileWriter` for persisting files.

# Operational Risks
- **Missing `excel_path`** – leads to `NullPointerException` or file write failures. *Mitigation*: validate property at startup; fallback to a default directory.
- **Large data sets** – building entire workbook in memory may cause `OutOfMemoryError`. *Mitigation*: stream rows using SXSSF for very large reports or paginate data.
- **Concurrent writes** – multiple threads invoking export methods could overwrite files with identical names. *Mitigation*: include unique timestamps or process IDs in filenames.
- **CSV quoting** – manual string concatenation may produce malformed CSV if data contains quotes or commas. *Mitigation*: use a CSV library (e.g., OpenCSV) to handle escaping.
- **Column auto‑size** – relies on the first row being present; empty sheets will skip sizing, potentially leaving unreadable columns. *Mitigation*: add guard for empty sheets or set default widths.

# Usage
```java
// Example from a Spring service or a main method
ExcelExport exporter = new ExcelExport();
List<ExportRecord> activesHol = fetchHolActives(); // obtain DTOs from upstream service
List<ExportRecord> activesSng = fetchSngActives();
...
String reportFile = exporter.exportDailyReport("2024-09", activesHol, activesSng,
                                            tollHol, tollSng, ipvProbe, eSimApis, usage);
System.out.println("Generated: " + reportFile);
```
To debug, set Log4j level to `DEBUG` for `com.tcl.move.service.ExcelExport` and verify the `excel_path` system property.

# Configuration
- **System Property**: `excel_path` – absolute path where all generated Excel/CSV files are written.
- **Log4j** configuration file (e.g., `log4j.properties`) must define the logger for `com.tcl.move.service.ExcelExport`.

# Improvements
1. **Streamed Workbook Generation** – replace `XSSFWorkbook` with `SXSSFWorkbook` for memory‑efficient handling of large reports.
2. **Robust CSV Generation** – adopt a dedicated CSV writer (e.g., OpenCSV) to correctly escape delimiters and quotes, reducing risk of malformed output.