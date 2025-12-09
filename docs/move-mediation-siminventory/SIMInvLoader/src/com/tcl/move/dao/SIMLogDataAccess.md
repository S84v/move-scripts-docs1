# Summary
`SIMLogDataAccess` provides DAO operations for the SIM Inventory loader’s audit and batch‑run tracking. It reads SQL statements from the `MOVEDAO` resource bundle, obtains Oracle connections via `JDBCConnection`, and inserts, updates, or queries the `MOVE_LOADER_LOG` and `MOVE_SIM_INVENTORY_FILE_AUDIT` tables. All actions are logged with Log4j and wrapped with custom exception handling.

# Key Components
- **Class `SIMLogDataAccess`**
  - `logger` – Log4j logger for traceability.
  - `daoBundle` – `ResourceBundle` loading `MOVEDAO.properties`.
  - `List<Timestamp> getLastLoaderRunData(String feed)` – Retrieves the most recent batch start/end timestamps for a given feed; returns `null` if the last status is “ATTN”.
  - `void updateLoaderRunData(String feed, Date startDate, Date endDate, String status)` – Inserts a new batch record (when `endDate` is `null`) or updates an existing one with end time and status.
  - `void insertIntoAudit(SIMInvAudit auditRecord)` – Persists a new audit row (`file_name`, `processing_start`, `remarks`).
  - `void updateAudit(SIMInvAudit auditRecord)` – Updates audit row with `record_count`, `processing_end`, and `remarks`.
  - `private String getStackTrace(Exception e)` – Utility to convert stack trace to a string for logging.

# Data Flow
| Method | Input | DB Interaction | Output / Side‑Effect |
|--------|-------|----------------|----------------------|
| `getLastLoaderRunData` | `feed` (String) | SELECT on `MOVE_LOADER_LOG` (Oracle) using parameters `feed`, `'SIM'` | `List<Timestamp>` (start, end) or `null` |
| `updateLoaderRunData` | `feed`, `startDate`, `endDate`, `status` | INSERT (if `endDate` = null) or UPDATE on `MOVE_LOADER_LOG` | DB row created/updated |
| `insertIntoAudit` | `SIMInvAudit` DTO | INSERT into `MOVE_SIM_INVENTORY_FILE_AUDIT` | New audit row |
| `updateAudit` | `SIMInvAudit` DTO | UPDATE on `MOVE_SIM_INVENTORY_FILE_AUDIT` (identified by `file_name` & `processing_start`) | Audit row modified |

All methods acquire a fresh Oracle connection via `JDBCConnection.getOracleConnection()`, close the connection in a `finally` block, and log each step.

# Integrations
- **`JDBCConnection`** – Centralized Oracle connection manager; provides `getOracleConnection()` and `closeOracleConnection()`.
- **`MOVEDAO.properties`** – Holds SQL statements referenced by keys: `get.dates`, `date.insert`, `date.update`, `audit.insert`, `audit.update`.
- **`SIMInvAudit` DTO** – Simple POJO containing audit fields (`fileName`, `startDate`, `endDate`, `recCount`, `remarks`).
- **Custom Exceptions** – `DBConnectionException`, `DBDataException` propagate error context to higher‑level job orchestration (e.g., the main loader driver).

# Operational Risks
- **Connection Leak** – If `closeOracleConnection()` fails, the connection may remain open. Mitigation: ensure `JDBCConnection` implements robust pool cleanup and monitor open sessions.
- **Hard‑coded Application Identifier** – `"SIM"` is embedded in SQL parameter binding; any rename requires code change. Mitigation: externalize as a configurable constant.
- **Null `endDate` handling** – Insert statement receives a `null` timestamp for `batch_end`; some Oracle drivers may reject explicit `null` binding. Verify driver behavior or use `setNull`.
- **Date Boundary Logic** – First‑run fallback sets start/end to 31 Dec 2014 23:59:59; if the system’s epoch changes, batch may miss data. Mitigation: make baseline date configurable.

# Usage
```java
// Example: retrieve last run timestamps
SIMLogDataAccess dao = new SIMLogDataAccess();
List<Timestamp> lastRun = dao.getLastLoaderRunData("SIM_ACTIVES");

// Example: start a new batch
Date start = new Date();
dao.updateLoaderRunData("SIM_ACTIVES", start, null, "START");

// After processing, close batch
Date end = new Date();
dao.updateLoaderRunData("SIM_ACTIVES", start, end, "SUCCESS");

// Audit record insertion
SIMInvAudit audit = new SIMInvAudit();
audit.setFileName("sim_actives_20251209.csv");
audit.setStartDate(new Timestamp(start.getTime()));
audit.setRemarks("Processing started");
dao.insertIntoAudit(audit);

// Later update audit
audit.setRecCount(12500);
audit.setEndDate(new Timestamp(end.getTime()));
audit.setRemarks("Processing completed");
dao.updateAudit(audit);
```
Run within the SIM Inventory loader job; ensure `MOVEDAO.properties` is on the classpath and Oracle JDBC driver is available.

# Configuration
- **System Properties / Env Vars** (populated by `MNAAS_ShellScript.properties`):
  - Oracle URL, user, password (used by `JDBCConnection`).
- **Resource Files**:
  - `MOVEDAO.properties` – SQL statements referenced by keys used in this DAO.
- **Log4j Configuration** – `log4j.properties` to control logging level and appenders.

# Improvements
1. **Refactor to Use a Connection Pool** – Replace manual open/close with a vetted pool (e.g., HikariCP) to improve performance and guarantee cleanup.
2. **Externalize Application Identifier** – Move `"SIM"` into a configurable constant or property to avoid hard‑coding and simplify reuse across environments.