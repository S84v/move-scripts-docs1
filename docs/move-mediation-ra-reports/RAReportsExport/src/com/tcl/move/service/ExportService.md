# Summary
`ExportService` is a production‑grade service class that orchestrates the generation of various Move‑Mediation revenue‑assurance reports (daily, monthly, RA‑specific, IPvProbe, PPU, E2E). It delegates the actual file creation to `ExcelExportDAO`, handles `FileException` conversion to `DatabaseException`, and logs failures via Log4j.

# Key Components
- **class `ExportService`**
  - `exportDailyReports(...)` – creates daily file‑wise, reject, and addon CSV reports.
  - `exportMonthlyReports(...)` – creates monthly file‑wise, reject, Geneva, and addon reports.
  - `exportRAMonthlyReports(...)` – creates RA monthly file, customer‑specific file, and RA reject ZIP.
  - `exportIPvProbeRAMonthlyReports(...)` – creates IPvProbe monthly Excel report.
  - `exportPPUReports(...)` – creates PPU subscription and usage CSVs for daily or monthly granularity.
  - `exportE2EReconReport(...)` – creates end‑to‑end reconciliation Excel report with rejects and outliers.
- **Dependencies**
  - `ExcelExportDAO exportDAO` – DAO responsible for low‑level file I/O (Excel/CSV generation, ZIP creation).
  - `Logger logger` – Log4j logger for error reporting.
  - Exceptions: `FileException` (DAO‑level), `DatabaseException` (service‑level).

# Data Flow
| Method | Input Parameters | DAO Calls (outputs) | Side Effects |
|--------|------------------|---------------------|--------------|
| `exportDailyReports` | Lists of `FileExportRecord`, `RejectExportRecord`, `AddonNotificationRecord`; `reportDate` | `exportFilewiseReport`, `exportRejectReport`, `exporDailyAddonReport` → creates `*.xlsx` and `*.csv` files in configured output directory | Files written to disk; on `FileException` logs and throws `DatabaseException`. |
| `exportMonthlyReports` | Lists of `FileExportRecord`, `RejectExportRecord`, `FileExportRecord` (Geneva), `AddonSummaryRecord`; `reportMonth` | `exportFileDateReport`, `exportRejectReport`, `exportFilewiseReport`, `exportMonthlyAddonReport` → creates monthly `*.xlsx`/`*.csv`. |
| `exportRAMonthlyReports` | Lists of `RAFileExportRecord` (monthly, cust), map of reject records, `month` | `exportRAMonthyReport` (twice), `exportRARejectReport` → creates monthly RA Excel files and a ZIP of rejects. |
| `exportIPvProbeRAMonthlyReports` | List of `IPvProbeExportRecord`, `reportMonth` | `exportNewIPvFilewiseReport` → creates IPvProbe `*.xlsx`. |
| `exportPPUReports` | Lists of `PPUSubsRecord`, `PPUUsageRecord`; `reportType` (`D`/`M`); `reportDate` | `exportSubsPPUReports`, `exportUsagePPUReports` → creates `*.csv` files for subs and usage. |
| `exportE2EReconReport` | `E2ERecord`, lists of `RejectRecord` & `OutlierRecord`; `reportMonth` | `exportE2EReconReport` → creates `E2E_*.xlsx`. |

All methods catch `FileException`, log the error, and re‑throw as `DatabaseException` to propagate to higher layers (e.g., batch driver).

# Integrations
- **DAO Layer**: `ExcelExportDAO` (same package) – performs actual Excel/CSV generation using Apache POI or similar libraries.
- **Batch Driver**: Invoked by `RAReportExport.main` (entry point) which assembles DTO lists from Hadoop/DB queries and calls the appropriate `ExportService` method.
- **Logging**: Log4j configuration (external `log4j.properties`) captures error logs.
- **File System**: Writes reports to paths constructed from `Constants` (e.g., `Constants.FILE_DAILY`, `Constants.REJECT_DAILY`), typically a shared NFS or HDFS mount used by downstream distribution scripts (mail, SCP).

# Operational Risks
- **File I/O Failure**: Disk full, permission issues, or network mount loss cause `FileException`. Mitigation: monitor filesystem health, enforce quota alerts, and ensure retry logic at batch level.
- **Data Volume Overflow**: Large record sets may exceed JVM heap or Excel row limits (65,536/1,048,576). Mitigation: stream writes, split files, or use CSV for massive datasets.
- **Exception Masking**: Converting `FileException` to `DatabaseException` loses original stack trace. Mitigation: include original exception as cause (`new DatabaseException(e)`).
- **Hard‑coded Constants**: File naming relies on `Constants`; any change requires code redeploy. Mitigation: externalize naming patterns to a properties file.

# Usage
```java
// Unit test / debug example
ExportService svc = new ExportService();
List<FileExportRecord> files = fetchFileRecords();          // mock or real DAO
List<RejectExportRecord> rejects = fetchRejectRecords();
List<AddonNotificationRecord> addons = fetchAddonRecords();
svc.exportDailyReports(files, rejects, addons, "2025-12-07");
```
In production, `RAReportExport` parses CLI args, builds DTO collections, and calls the appropriate method.

# Configuration
- **Constants.java** – defines directory prefixes and file name templates (`DAILY_FILE`, `FILE_DAILY`, etc.).
- **log4j.properties** – controls logging level and appenders for `ExportService`.
- **Environment** – JVM must have write permission to the output directory defined by constants; optionally `HADOOP_CONF_DIR` for source data extraction (outside this class).

# Improvements
1. **Preserve Exception Cause** – change `throw new DatabaseException(e.getMessage());` to `throw new DatabaseException(e.getMessage(), e);` to retain stack trace.
2. **Streamlined File Generation** – replace in‑memory Excel creation with Apache POI SXSSF (streaming) or direct CSV for large reports to reduce memory pressure and avoid row‑limit failures.