# Summary
`GenevaFileDataAccess` generates Geneva‑compatible event and control files for the Move‑mediation billing reconciliation workflow. It formats header/footer sections, inserts SIM or usage event records from `EventFile` DTOs, writes the files to a configurable directory, and logs errors via Log4j. The produced file names are returned for inclusion in control files.

# Key Components
- **Class `GenevaFileDataAccess`**
  - `writeEventFile(String strNow, String eventType, String serialNumber, List<EventFile> records)`: builds event file, selects SIM or usage insertion, returns file name.
  - `writeControlFile(String strNow, String eventType, String serialNumber, String eventFileName)`: builds control file referencing the event file.
  - `insertSIMEvents(FileWriter writer, List<EventFile> records)`: formats and writes SIM event lines.
  - `insertUsageEvents(FileWriter writer, List<EventFile> records)`: formats and writes usage event lines (filters out zero usage).
  - `getStackTrace(Exception e)`: utility to capture stack trace as string.
- **Dependencies**
  - `org.apache.log4j.Logger` for logging.
  - `com.tcl.move.constants.Constants` for date formats and event type literals.
  - `com.tcl.move.dto.EventFile` DTO containing all event attributes.
  - `com.tcl.move.exceptions.FileException` custom checked exception.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | `strNow`, `eventType`, `serialNumber`, `List<EventFile>` | Build header array, compute line count, select insertion method based on first record’s `eventType`. | Event file written to `${MNAAS_Billing_Geneva_FilesPath}`; file name returned. |
| 2 | Same parameters + generated `eventFileName` | Write static control file header, body referencing event file, static footer. | Control file written to same directory. |
| 3 | `EventFile` objects | `insertSIMEvents` → concatenates fields per Geneva spec.<br>`insertUsageEvents` → formats numeric fields, applies DecimalFormat, skips records with `actualUsage <= 0`. | Event lines appended to open `FileWriter`. |
| 4 | Exceptions | Captured, logged, wrapped in `FileException`. | Propagation of `FileException` to caller. |

External services: none (file system only). No DB or queue interaction in this class.

# Integrations
- **Caller components** (e.g., batch jobs or service layers) invoke `writeEventFile` and `writeControlFile` after retrieving/transforming data via DAOs such as `DataUsageDataAccess`.
- **Constants** provide date patterns (`EVENT_DATE_FORMAT`, `PERIOD_DATE_FORMAT`) and event type identifiers (`SIM`, `USAGE`).
- **EventFile DTO** populated by upstream processing (e.g., `DataUsageDataAccess` or other business logic).
- **FileException** handled by higher‑level orchestration to trigger retries or alerts.

# Operational Risks
- **Hard‑coded header/footer arrays**: any change in Geneva spec requires code redeployment. *Mitigation*: externalize to a properties file.
- **Assumption that `eventFileRecords` is non‑empty** (accesses index 0). Empty list causes `IndexOutOfBoundsException`. *Mitigation*: validate list size before use.
- **File path reliance on system property `MNAAS_Billing_Geneva_FilesPath`**; missing or incorrect value leads to `FileNotFoundException`. *Mitigation*: validate at startup, fallback to default directory, and monitor property.
- **No charset specification**: uses platform default, may produce non‑ASCII characters. *Mitigation*: enforce UTF‑8 or explicit ASCII encoding via `OutputStreamWriter`.
- **Potential resource leak if `FileWriter` construction fails** (writer remains null, but close in finally may still be called). Current code handles null check; ensure no early return before finally.

# Usage
```java
// Example unit‑test / debugging snippet
GenevaFileDataAccess dao = new GenevaFileDataAccess();
List<EventFile> events = loadTestEvents(); // populate DTOs
String timestamp = new SimpleDateFormat("yyyyMMddHHmmss").format(new Date());
String eventFile = dao.writeEventFile(timestamp, Constants.USAGE, "001", events);
dao.writeControlFile(timestamp, Constants.USAGE, "001", eventFile);
```
Run within the application context where `MNAAS_Billing_Geneva_FilesPath` is set (e.g., `-DMNAAS_Billing_Geneva_FilesPath=/opt/geneva/files/`).

# configuration
- **System Property** `MNAAS_Billing_Geneva_FilesPath` – directory path for generated files.
- **Constants** (via `Constants.java`):
  - `EVENT_DATE_FORMAT` – pattern for event timestamps.
  - `PERIOD_DATE_FORMAT` – pattern for start/end period dates.
  - `SIM`, `USAGE` – literal identifiers used to select insertion logic.

# Improvements
1. **Externalize header/footer definitions** to a configuration file (properties or JSON) to allow runtime updates without code changes.
2. **Add input validation**: check for empty `eventFileRecords`, verify required fields in `EventFile`, and enforce explicit UTF‑8 encoding when writing files.