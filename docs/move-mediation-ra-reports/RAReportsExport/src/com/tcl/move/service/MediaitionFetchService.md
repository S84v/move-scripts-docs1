# Summary  
`MediaitionFetchService` is a production‑grade service that encapsulates all data‑retrieval operations required by the Move‑Mediation revenue‑assurance batch jobs. It delegates raw SQL/Hadoop queries to `MediaitionFetchDAO`, translates low‑level `DBConnectionException` / `DBDataException` into a unified `DatabaseException`, and logs failures. The service supplies DTO collections for daily, monthly, RA‑specific, IPvProbe, PPU, Add‑on, and E2E reports.

# Key Components  
- **Class `MediaitionFetchService`** – façade for data access.  
- **`getDailyFileRecords(Date)`** – returns `List<FileExportRecord>` for a single day.  
- **`getDailyRejectRecords(Date)`** – returns `List<RejectExportRecord>` for a single day.  
- **`getMonthlyFileRecords(Date, Date)`** – returns file counts for a date range (Teleena/Hadoop).  
- **`getGenevaFileRecords(Date, Date)`** – returns file counts for Geneva source.  
- **`getMonthlyRejectRecords(Date, Date)`** – returns reject counts for a date range.  
- **`getRAMonthlyReport(String)`**, **`getRACustMonthlyReport(String)`**, **`getRAMonthlyRejReport(String)`** – RA‑specific monthly DTOs (including reject map).  
- **`getIpvProbeMonthlyReport(String, String, String)`** – IPvProbe DTOs for a month and optional date window.  
- **`getUsagePPUReport(Date, String)`**, **`getSubsPPUReport(Date, String)`** – PPU usage/subscription DTOs.  
- **`getDailyAddonRecords(Date)`**, **`getMonthlyAddonReport(Date)`** – Add‑on notification and summary DTOs.  
- **`getE2ERecord(String, String, Date)`** – aggregates file‑wise, reconciliation, and Geneva counts into a single `E2ERecord`.  
- **`getOutliers(String, String)`**, **`getRejects(String)`**, **`getLateLandingCDRs(String)`** – placeholders returning empty collections.  
- **`private static getStackTrace(Exception)`** – utility to convert stack trace to `String` for logging.

# Data Flow  
| Method | Input(s) | DAO Call | Output | Side Effects |
|--------|----------|----------|--------|--------------|
| `getDailyFileRecords` | `Date exportDate` | `mediationDAO.fetchDailyFileCounts` | `List<FileExportRecord>` | DB read, logs on error |
| `getDailyRejectRecords` | `Date exportDate` | `mediationDAO.fetchDailyRejectCounts` | `List<RejectExportRecord>` | DB read, logs |
| `getMonthlyFileRecords` | `Date startDate, endDate` | `mediationDAO.fetchMonthlyFileCounts` | `List<FileExportRecord>` | DB read |
| `getGenevaFileRecords` | `Date startDate, endDate` | `mediationDAO.fetchMonthlyGenFileCounts` | `List<FileExportRecord>` | DB read |
| `getMonthlyRejectRecords` | `Date startDate, endDate` | `mediationDAO.fetchMonthlyRejectCounts` | `List<RejectExportRecord>` | DB read |
| `getRAMonthlyReport` | `String month` | `mediationDAO.fetchRAMonthlyReport` | `List<RAFileExportRecord>` | DB read |
| `getRACustMonthlyReport` | `String month` | `mediationDAO.fetchRACustMonthlyReport` | `List<RAFileExportRecord>` | DB read |
| `getRAMonthlyRejReport` | `String month` | `mediationDAO.fetchRAMonthlyRejectReport` | `Map<String, List<RAFileExportRecord>>` | DB read |
| `getIpvProbeMonthlyReport` | `String month, startDate, endDate` | `mediationDAO.fetchIpvProbeMonthlyReport` | `List<IPvProbeExportRecord>` | DB read |
| `getUsagePPUReport` | `Date reportDate, String repType` | `mediationDAO.fetchPPUUsageReport` | `List<PPUUsageRecord>` | DB read |
| `getSubsPPUReport` | `Date reportDate, String repType` | `mediationDAO.fetchPPUSubsReport` | `List<PPUSubsRecord>` | DB read |
| `getDailyAddonRecords` | `Date reportDate` | `mediationDAO.fetchDailyAddonReport` | `List<AddonNotificationRecord>` | DB read |
| `getMonthlyAddonReport` | `Date startDate` | `mediationDAO.fetchMonthlyAddonReport` | `List<AddonSummaryRecord>` | DB read |
| `getE2ERecord` | `String fileMonth, month, Date startDate` | 3 DAO calls (`fetchfileWiseCounts`, `fetchReconCounts`, `fetchGenevaCounts`) | `E2ERecord` | DB reads, date calculations |
| Placeholder methods | – | – | empty collections | – |

All methods catch `DBConnectionException` and `DBDataException`, log the error via Log4j, and re‑throw a generic `DatabaseException`. Unexpected exceptions are also logged with full stack trace.

# Integrations  
- **DAO Layer**: `MediaitionFetchDAO` (direct JDBC/Hadoop queries).  
- **Batch Orchestrator**: `ExportService` / `RAReportExport` invoke these service methods to populate DTOs before passing them to `ExcelExportDAO`.  
- **Logging**: Log4j configuration (external `log4j.properties`).  
- **Exception Propagation**: Upstream components handle `DatabaseException` to trigger alerts via `MailService` or retries.  

# Operational Risks  
- **Database Connectivity** – transient DB outages cause `DatabaseException`; mitigation: retry logic in calling batch or connection pool configuration.  
- **Uncaught Runtime Exceptions** – only logged; ensure upstream catches `DatabaseException` to avoid silent job failure.  
- **Large Result Sets** – methods return full `List<>`; risk of OOM for high‑volume months. Mitigation: paginate in DAO or stream results.  
- **Hard‑coded Date Formats** – reliance on `Constants.YM_HYPHEN_FORMAT`/`YMD_HYPHEN_FORMAT`; any format change requires code update.  

# Usage  
```java
// Example unit test / debugging snippet
MediaitionFetchService svc = new MediaitionFetchService();
Date today = new SimpleDateFormat("yyyy-MM-dd").parse("2024-09-30");
try {
    List<FileExportRecord> daily = svc.getDailyFileRecords(today);
    System.out.println("Rows: " + daily.size());
} catch (DatabaseException e) {
    e.printStackTrace();
}
```
In production the service is instantiated by `ExportService