# Summary
`Utils` is a production‑grade helper class for the Move‑Mediation RA Reports batch. It provides date‑range calculations, file‑zipping, and file‑move operations used by the export workflow. Errors are wrapped in the custom `FileException`. A `MailService` instance is created but never used in the current code.

# Key Components
- **class `Utils`**
  - `static Date getFirstDayofTheMonth(String monthYear)` – parses `yyyy‑MM` and returns the first instant of that month.
  - `static Date getLastDayofTheMonth(String monthYear)` – parses `yyyy‑MM` and returns the last instant of that month.
  - `static Date getYesterday()` – returns yesterday at 00:00:00.
  - `static Date getFirstDayofLastMonth()` – first day of the previous month at 00:00:00.
  - `static Date getLastDayofLastMonth()` – last day of the previous month at 23:59:59.
  - `static Date getFirstDay(Date date)` – normalises any date to the first day of its month at 00:00:00.
  - `static Date getLastDay(Date date)` – normalises any date to the last day of its month at 23:59:59.
  - `static void zipFile(String fileName, String zipFileName)` – creates a ZIP archive containing a single file; throws `FileException` on I/O errors.
  - `static void moveFiles(String sourcePath, String destPath)` – moves all files from a source directory (or a single file) to a destination directory, overwriting existing files; throws `FileException` on failure.
- **field**
  - `MailService mailService = new MailService();` – instantiated but not referenced.

# Data Flow
| Method | Input(s) | Output / Side‑Effect | External Interaction |
|--------|----------|----------------------|-----------------------|
| `getFirstDayofTheMonth` / `getLastDayofTheMonth` | `String monthYear` (`yyyy‑MM`) | `java.util.Date` | None |
| `getYesterday`, `getFirstDayofLastMonth`, `getLastDayofLastMonth` | none | `Date` | None |
| `getFirstDay`, `getLastDay` | `Date` (nullable) | `Date` | None |
| `zipFile` | `fileName` (path to existing file), `zipFileName` (target zip path) | Creates/overwrites zip file on filesystem | Filesystem I/O; may propagate `FileException` |
| `moveFiles` | `sourcePath` (file or directory), `destPath` (directory path ending with separator) | Moves files on filesystem, overwriting if present | Filesystem I/O; may propagate `FileException` |

# Integrations
- **MailService**: imported and instantiated; potential future use for error notifications (currently unused).
- **FileException**: custom exception type used throughout to surface I/O problems to callers (e.g., batch orchestrator).
- **Batch workflow**: other services (`MediaitionFetchService`, `SCPService`, `MailService`) call `Utils` for date calculations and file handling when generating and transferring reports.

# Operational Risks
- **Date parsing errors** – throws generic `Exception`; could cause batch failure if input format deviates. *Mitigation*: validate format before calling or use a dedicated parser.
- **Hard‑coded buffer size & stream handling** – potential memory pressure on large files. *Mitigation*: stream with larger buffer or use NIO channels.
- **`moveFiles` does not verify destination directory existence** – may fail with `NoSuchFileException`. *Mitigation*: ensure dest directory is created or check before move.
- **Unused `MailService` instance** – unnecessary object creation, minor overhead. *Mitigation*: remove or integrate for error alerts.

# Usage
```java
// Example: zip a report and move it to the archive directory
String reportPath = "/data/reports/dailyReport.csv";
String zipPath    = "/data/archive/dailyReport.zip";
String archiveDir = "/data/archive/";

// Zip the file
try {
    Utils.zipFile(reportPath, zipPath);
} catch (FileException fe) {
    // handle zip failure
}

// Move the zip to archive (overwrites if exists)
try {
    Utils.moveFiles(zipPath, archiveDir);
} catch (FileException fe) {
    // handle move failure
}
```
Debug by stepping through each static method; watch for `FileException` messages for I/O root cause.

# Configuration
- No external configuration is read directly by `Utils`. It relies on caller‑provided paths and date strings.
- `MailService` (instantiated) reads system properties `error_mail_to`, `ra_mail_from`, `mail_host` when used elsewhere.

# Improvements
1. **Replace generic `Exception` in date parsers with a specific checked exception** (e.g., `InvalidMonthFormatException`) and validate input format using a regex or `DateTimeFormatter`.
2. **Add destination‑directory validation and creation in `moveFiles`** to guarantee successful moves and reduce `FileException` noise.