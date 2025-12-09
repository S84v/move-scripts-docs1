# Summary
`ExcelExportDAO` generates various Excel (XLSX) and CSV reports for the Move‑Mediation RA system. It consumes in‑memory DTO collections, formats them into worksheets with headers, applies styling/merging, auto‑sizes columns, writes the files to a configurable output directory, and optionally archives/ zips reject reports.

# Key Components
- **class ExcelExportDAO**
  - `exportFilewiseReport(List<FileExportRecord>, String reportType, String fileName)`: Daily/Monthly file‑level count report (incl. Geneva data).
  - `exportRejectReport(List<RejectExportRecord>, String reportType, String fileName)`: Daily/Monthly reject report (CSV‑style Excel).
  - `exportFileDateReport(List<FileExportRecord>, String fileName)`: File‑date summary report.
  - `exportRAMonthyReport(List<RAFileExportRecord>, String reportType, String fileName)`: RA monthly CDR/usage matrix with optional customer columns.
  - `exportRARejectReport(Map<String,List<RAFileExportRecord>>, String fileName, String zipFileName)`: Multi‑sheet reject reports, writes XLSX, zips, moves to backup.
  - `exportNewIPvFilewiseReport(List<IPvProbeExportRecord>, String fileName)`: IPv probe file‑wise report with merged rows for same file.
  - `exportSubsPPUReports(List<PPUSubsRecord>, String reportType, String fileName)`: CSV export for subscription‑PPU summary.
  - `exportUsagePPUReports(List<PPUUsageRecord>, String reportType, String fileName)`: CSV export for usage‑PPU summary.
  - `exporDailyAddonReport(List<AddonNotificationRecord>, String fileName)`: CSV export for daily addon notifications.
  - `exportMonthlyAddonReport(List<AddonSummaryRecord>, String fileName)`: Excel export for monthly addon counts.
  - `exportE2EReconReport(E2ERecord, List<RejectRecord>, List<OutlierRecord>, List<OutlierRecord>, List<OutlierRecord>, String fileName)`: Populates a pre‑formatted template with E2E reconciliation data.
  - `autoSizeColumns(XSSFWorkbook)`: Helper to auto‑size all columns in each sheet.
  - `getStackTrace(Exception)`: Utility to capture stack trace as string.

# Data Flow
| Method | Input | Processing | Output / Side‑Effect |
|--------|-------|------------|----------------------|
| `export*Report` (all) | DTO lists (FileExportRecord, RejectExportRecord, etc.) | Build XSSFWorkbook, create sheets, style headers, fill rows, merge cells where needed, auto‑size columns | Writes XLSX/CSV to `System.getProperty("extract_path") + fileName` |
| `exportRARejectReport` | Map of category → List<RAFileExportRecord>, fileName, zipFileName | Same as above + `Utils.zipFile` to create zip, `Utils.moveFiles` to backup directory | XLSX, zip archive, moved original file |
| `exportE2EReconReport` | Single `E2ERecord` (template‑based), reject/outlier lists | Load template workbook from hard‑coded path, write values into specific rows, auto‑size, write output file | XLSX written to extract path |
| `autoSizeColumns` | XSSFWorkbook | Iterate sheets, first row cells → `sheet.autoSizeColumn` | Adjusted column widths in workbook |

External services:
- `Utils.zipFile` – creates zip archive.
- `Utils.moveFiles` – moves file to backup location.
- `Logger` – logs progress and errors.
- No DB/queue interaction; all data supplied by upstream services.

# Integrations
- **DAO Consumers**: Service layer classes that assemble DTO collections from Hadoop/RA processing pipelines and invoke the appropriate `export*` method.
- **Utils**: Shared utility class for file compression and movement.
- **Constants**: Central enum of sheet names, report type strings (e.g., `Constants.GENEVA_FILE`, `Constants.DAILY`).
- **Template File**: `exportE2EReconReport` expects a static Excel template at `C:\Anu Office\RA Reports\Template\E2E Recon Summary Report - Jan 2023.xlsx`.

# Operational Risks
- **Hard‑coded template path** – fails if file moved; mitigate by externalizing path to config.
- **Large DTO lists** – memory pressure when building workbook; mitigate by streaming or chunked writes.
- **Cell merging logic** in `exportFilewiseReport` and `exportNewIPvFilewiseReport` may leave orphan merged regions on errors; ensure cleanup in finally block.
- **File system permissions** – write to `extract_path` and backup path must be writable; monitor OS permissions.
- **Zip/Move failures** not propagated (only logged); consider propagating as `FileException` to trigger retry.

# Usage
```java
// Example from a unit test or service
ExcelExportDAO dao = new ExcelExportDAO();
List<FileExportRecord> records = fetchFromService();
dao.exportFilewiseReport(records, Constants.GENEVA_FILE, "geneva_report.xlsx");

// Debug: set system properties before JVM start
System.setProperty("extract_path", "/opt/reports/");
System.setProperty("backup_path", "/opt/reports/backup/");
```
Run with appropriate classpath containing Apache POI, Log4j, and project DTOs.

# Configuration
- `System.getProperty("extract_path")` – directory for generated reports.
- `System.getProperty("backup_path")` – directory where original XLSX is moved after zipping.
- `Constants` class provides report type identifiers and sheet names.
- No external config files referenced directly.

# Improvements
1. **Externalize template location** – replace hard‑coded path with a configurable property (e.g., `e2e.template.path`).
2. **Stream large worksheets** – use `SXSSFWorkbook` (Apache POI streaming) to reduce heap usage for massive DTO collections.