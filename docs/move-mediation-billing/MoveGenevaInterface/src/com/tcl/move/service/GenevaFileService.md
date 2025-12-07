# Summary
`GenevaFileService` orchestrates the creation of Geneva control and event files for SIM and usage charge data, persists related metadata to Oracle, and records RA records when applicable. It is invoked by the batch driver (`GenevaLoader`) during the Move‑Geneva mediation process to deliver billing events to the Geneva server.

# Key Components
- **class `GenevaFileService`**
  - `loadGenevaFiles(List<EventFile> eventFileRecords, List<RARecord> allRARecords, Long secs, String eventType, Date startDate)`: core method that generates file names, writes event/control files, inserts processing metadata, and handles RA records.
  - `logger`: Log4j logger for operational tracing.
  - `genevaFileDAO`: DAO (`GenevaFileDataAccess`) responsible for file I/O.
  - `oracleDAO`: DAO (`OracleDataAccess`) responsible for serial number generation and DB inserts.
- **Exception handling**
  - Catches `DBDataException`, `DBConnectionException`, `FileException` and re‑throws as `DatabaseException` to unify error propagation.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | `eventFileRecords` (list of `EventFile`), `eventType`, current timestamp | Generate timestamp string (`strNow`) using `Constants.FILE_NAME_DATE_FORMAT` | `strNow` |
| 2 | `oracleDAO.getUniqueSerialNumber()` | Retrieve unique serial number from Oracle sequence/table | `serialNumber` |
| 3 | `genevaFileDAO.writeEventFile(strNow, eventType, serialNumber, eventFileRecords)` | Serialize `eventFileRecords` to Geneva‑formatted event file | `eventFileName` (path) |
| 4 | `genevaFileDAO.writeControlFile(strNow, eventType, serialNumber, eventFileName)` | Create corresponding control file | Control file on filesystem |
| 5 | `oracleDAO.insertInTable(eventFileRecords, secs, eventType)` | Persist event metadata (e.g., file reference, processing ID) | DB rows in processing table |
| 6 | Optional `oracleDAO.insertRARecords(allRARecords, secs, startDate, eventFileName)` | Persist RA records if provided | DB rows in RA table |
| 7 | Exceptions (`DBDataException`, `DBConnectionException`, `FileException`) | Log error, wrap and re‑throw as `DatabaseException` | Propagation to caller (`GenevaLoader`) |

# Integrations
- **File System**: Writes event and control files to a directory expected by the Geneva server (path defined inside `GenevaFileDataAccess`).
- **Oracle Database**: Uses `OracleDataAccess` for:
  - Serial number generation (`getUniqueSerialNumber`).
  - Inserting event processing metadata (`insertInTable`).
  - Inserting RA records (`insertRARecords`).
- **DAO Layer**: `GenevaFileDataAccess` abstracts file I/O; `OracleDataAccess` abstracts DB operations.
- **Batch Driver**: Invoked by `GenevaLoader.main` as part of the mediation batch workflow.
- **Exception Model**: Converts lower‑level DB/file exceptions into `DatabaseException` for uniform handling upstream.

# Operational Risks
- **File I/O Failure**: Disk full or permission issues cause `FileException`; mitigated by monitoring disk usage and ensuring proper ACLs.
- **Serial Number Collision**: Failure to obtain a unique serial number leads to duplicate file names; mitigate by using Oracle sequence with proper locking.
- **Partial Persistence**: If DB insert succeeds but file write fails (or vice‑versa), data may become inconsistent; mitigate by implementing transactional compensation or two‑phase commit.
- **Uncaught Runtime Exceptions**: Only checked exceptions are wrapped; unexpected `RuntimeException` could terminate the batch; mitigate by adding a generic catch block with logging.

# Usage
```java
// Example unit test / debug snippet
List<EventFile> events = ...;          // populate with test data
List<RARecord> raRecords = ...;        // optional
Long secs = 12345L;
String eventType = "SIM";              // or "Usage"
Date startDate = new SimpleDateFormat("yyyy-MM-dd").parse("2024-01-01");

GenevaFileService service = new GenevaFileService();
try {
    service.loadGenevaFiles(events, raRecords, secs, eventType, startDate);
    System.out.println("Files generated and DB updated successfully.");
} catch (DatabaseException e) {
    e.printStackTrace();
}
```
Run within the application server or via the `GenevaLoader` batch entry point.

# configuration
- **Environment Variables / System Properties**
  - None referenced directly; configuration is encapsulated in DAO implementations.
- **Config Files**
  - `Constants.FILE_NAME_DATE_FORMAT` (e.g., `yyyyMMddHHmmss`) defined in `com.tcl.move.constants.Constants`.
  - DAO classes read DB connection details from the application’s datasource configuration (typically `datasource.xml` or JNDI).

# Improvements
1. **Transactional Consistency**: Wrap file writes and DB inserts in a single transaction (e.g., using a two‑phase commit or compensating rollback) to avoid partial state on failure.
2. **Enhanced Exception Mapping**: Add a generic `catch (Exception e)` block to log and re‑throw as `DatabaseException` to prevent uncaught runtime errors from aborting the batch.