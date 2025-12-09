# Summary
`ExcelExport` generates Excel workbooks for daily and monthly KLM reports. It builds three worksheets (Summary, In‑Bundle Usage, Out‑of‑Bundle Usage) from in‑memory DTO collections, applies basic styling, auto‑sizes columns, and writes the file to a path defined by the `excel_path` system property. Errors are logged with full stack traces.

# Key Components
- **class `ExcelExport`**
  - `autoSizeColumns(XSSFWorkbook)`: iterates all sheets, auto‑sizes each column based on the first row’s cells.
  - `exportKLMReport(String monthYear, Map<Integer,KLMRecord> klmSummary, Map<String,List<KLMRecord>> klmList)`: orchestrates workbook creation, sheet population, styling, column sizing, and file output.
  - `getStackTrace(Exception)`: utility to convert an exception stack trace to a `String` for logging.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | `monthYear`, `klmSummary` (Map<Integer,KLMRecord>), `klmList` (Map<String,List<KLMRecord>> ) | Populate three sheets with header rows and data rows using POI APIs. | In‑memory `XSSFWorkbook`. |
| 2 | Workbook from step 1 | `autoSizeColumns` adjusts column widths. | Modified workbook. |
| 3 | System property `excel_path` | `FileOutputStream` writes workbook to `excel_path + "KLM Report - " + monthYear + ".xlsx"`. | Physical `.xlsx` file on filesystem. |
| 4 | Exceptions during write | Logged via Log4j with full stack trace. | Log entry; no retry. |

# Integrations
- **Calling component**: `ReportGenerator` (or other service) invokes `exportKLMReport` after retrieving data from the database.
- **External libraries**: Apache POI (`org.apache.poi.*`) for Excel handling; Log4j for logging.
- **Configuration source**: System property `excel_path` supplied at JVM launch (e.g., `-Dexcel_path=/opt/reports/`).

# Operational Risks
- **File I/O failure** (disk full, permission error) → logged but not retried; may cause missing reports. *Mitigation*: pre‑flight check of path, monitor disk usage, implement retry/back‑off.
- **Memory pressure** for large datasets → entire workbook held in memory. *Mitigation*: stream API (`SXSSFWorkbook`) for very large reports.
- **Hard‑coded column range** (loops `i = 1 to 3` for summary) assumes exactly three summary rows; mismatch leads to `NullPointerException`. *Mitigation*: iterate over `klmSummary.keySet()` or validate size.
- **System property missing** → `NullPointerException` on `filePath`. *Mitigation*: validate `excel_path` at startup, provide default.

# Usage
```java
// Example unit‑test or debug snippet
Map<Integer, KLMRecord> summary = loadSummary();          // populate from DB
Map<String, List<KLMRecord>> details = loadDetails();    // "IN" and "OUT" lists
ExcelExport exporter = new ExcelExport();
exporter.exportKLMReport("2024-09", summary, details);
```
Run the application with:
```
java -Dexcel_path=/data/reports/ -cp <classpath> com.tcl.move.main.ReportGenerator
```

# Configuration
- **System property** `excel_path` – directory where the generated Excel file is written. Must end with a file‑separator or be concatenated correctly.
- No external config files referenced directly in this class.

# Improvements
1. **Replace `XSSFWorkbook` with `SXSSFWorkbook`** for streaming write to reduce heap consumption on large reports.
2. **Validate inputs and configuration**: add null checks for `excel_path`, `klmSummary`, `klmList`; throw a custom `FileException` on validation failure to allow upstream handling.