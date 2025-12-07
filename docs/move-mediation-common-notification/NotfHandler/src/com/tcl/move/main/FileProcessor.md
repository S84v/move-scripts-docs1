# Summary
`FileProcessor` reads CSV‑formatted files for SIM swap, MSISDN swap, and status‑change events, maps each line to a `NotificationRecord`, enriches static fields, and persists the record via `RawTableDataAccess.insertInRawTable`. It is invoked by downstream batch jobs that supply the source files.

# Key Components
- **Class `FileProcessor`**
  - `RawTableDataAccess dao` – DAO instance for raw table writes.
  - `processSIMSwap(File simSwapFile)` – parses SIM‑swap CSV, builds `NotificationRecord`, sets `Constants.SIM_SWAP` name, defaults `swapSource` when missing, writes to DB.
  - `processMSISDNSwap(File msisdnSwapFile)` – identical logic for MSISDN‑swap CSV, uses `Constants.MSISDN_SWAP`.
  - `processStatusChange(File statusChangeFile)` – parses status‑change CSV, uses `Constants.CHANGE_STATUS`, leaves result and swapSource empty.
  - Each method handles I/O with `BufferedReader`, splits lines on commas, and closes the reader in a `finally` block.

# Data Flow
- **Input:** Local file system CSV files (`File` objects) supplied to the three public methods.
- **Processing:** Line‑by‑line read → `String.split(",")` → populate `NotificationRecord` fields.
- **Output / Side Effects:** Calls `dao.insertInRawTable(rec)` → persists record to the raw notification table (Hive/Oracle). No return value.
- **External Services:** `RawTableDataAccess` (DB access layer). No messaging or external APIs.

# Integrations
- **Constants:** `com.tcl.move.constants.Constants` provides notification type strings.
- **DTO:** `com.tcl.move.dto.NotificationRecord` – data carrier for DB insert.
- **DAO:** `com.tcl.move.dao.RawTableDataAccess` – encapsulates JDBC/Hive interaction.
- **Batch Orchestrator:** Expected to instantiate `FileProcessor` and invoke the appropriate method per scheduled job.

# Operational Risks
- **Unvalidated CSV format:** IndexOutOfBoundsException if a line has fewer columns than expected. *Mitigation:* Add length checks before field access.
- **Resource leak on null reader:** `br.close()` called without null check may cause NPE if file open fails. *Mitigation:* Guard with `if (br != null)`.
- **Silent failure:** Exceptions are only printed (`e.printStackTrace()`), not propagated or logged to monitoring. *Mitigation:* Replace with structured logging and rethrow custom exceptions (`DatabaseException`, `ParseException`).
- **Hard‑coded column indices:** Changes in source file schema require code changes. *Mitigation:* Externalize mapping configuration.

# Usage
```java
FileProcessor processor = new FileProcessor();
processor.processSIMSwap(new File("/data/incoming/sim_swap_20251205.csv"));
processor.processMSISDNSwap(new File("/data/incoming/msisdn_swap_20251205.csv"));
processor.processStatusChange(new File("/data/incoming/status_change_20251205.csv"));
```
Run within a Java application or batch job; ensure classpath includes DAO, DTO, and constants packages.

# Configuration
- No environment variables referenced directly.
- Relies on `RawTableDataAccess` configuration (JDBC URL, credentials) defined elsewhere (typically in a properties file or Spring context).
- `Constants` class must be populated with correct notification name strings.

# Improvements
1. **Robust parsing:** Implement CSV parser (e.g., Apache Commons CSV) with header validation and safe field extraction.
2. **Error handling & logging:** Replace `printStackTrace` with a logging framework (SLF4J/Logback) and throw domain‑specific exceptions to enable retry or alerting.