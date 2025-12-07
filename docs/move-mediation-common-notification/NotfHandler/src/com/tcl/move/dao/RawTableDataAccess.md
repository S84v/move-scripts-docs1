# Summary
`RawTableDataAccess` is a DAO that persists various notification‑related DTOs into Hive or Oracle tables used by the Move Mediation Notification Handler (MNAAS). It converts in‑memory records (e.g., `NotificationRecord`, `LocationUpdateRecord`, `OCSRecord`, etc.) into parameterised INSERT statements, executes them via JDBC, logs activity, and wraps any failure in a `DatabaseException` with error codes from `Constants`.

# Key Components
- **Class `RawTableDataAccess`**
  - `insertInRawTable(NotificationRecord)` – writes successful notification events to Hive raw table.
  - `insertInRawFail(NotificationRecord)` – writes failed notification events to Oracle reject table.
  - `insertInLocationRawTable(LocationUpdateRawRecord)` – writes minimal location update data to Hive raw table.
  - `insertInLocationRawTable(LocationUpdateRecord)` – writes full location update data to Hive table plus custom attributes to a secondary attribute table.
  - `insertInOCSRawTable(OCSRecord)` – writes OCS bypass records to Hive.
  - `insertInESimRawTable(ESimRecord)` – writes eSIM provisioning records to Hive.
  - `insertInSIMIMEIRawTable(SimImeiRecord)` – writes SIM‑IMEI change records to Hive.
  - `getStackTrace(Exception)` – utility to convert an exception stack trace to a `String` for logging.

# Data Flow
| Method | Input DTO | Target DB | Table (via SQL key) | Side Effects |
|--------|-----------|-----------|---------------------|--------------|
| `insertInRawTable` | `NotificationRecord` | Hive | `insert.raw` | Log INFO/ERROR, throw `DatabaseException` on failure |
| `insertInRawFail` | `NotificationRecord` (with fail code/reason) | Oracle | `insert.raw.reject` | Same as above |
| `insertInLocationRawTable(LocationUpdateRawRecord)` | `LocationUpdateRawRecord` | Hive | `insert.location.raw` | Same |
| `insertInLocationRawTable(LocationUpdateRecord)` | `LocationUpdateRecord` | Hive | `insert.location` + `insert.location.attr` | Inserts main row then iterates over `customAttributes` map for attribute rows |
| `insertInOCSRawTable` | `OCSRecord` | Hive | `insert.ocs` | Same |
| `insertInESimRawTable` | `ESimRecord` | Hive | `insert.esim` | Same |
| `insertInSIMIMEIRawTable` | `SimImeiRecord` | Hive | `insert.simimei` | Same |

All methods acquire a JDBC `Connection` via `JDBCConnection`, prepare a `PreparedStatement`, bind DTO fields, execute `executeUpdate()`, close resources in a `finally` block, and log stack traces on exceptions.

# Integrations
- **`JDBCConnection`** – provides `getConnectionHive()` and `getConnectionOracle()`; reads connection parameters from system properties.
- **`Constants`** – supplies error‑code identifiers (`ERROR_RAW`, `ERROR_FAIL`) for `DatabaseException`.
- **`ResourceBundle` (`MOVEDAO`)** – external properties file containing the actual INSERT SQL strings referenced by keys such as `insert.raw`, `insert.location`, etc.
- **Log4j** – used for operational logging.
- **DTO classes** (`NotificationRecord`, `LocationUpdateRecord`, etc.) – supplied by upstream processors (e.g., Kafka consumers, service handlers).

# Operational Risks
- **Resource leakage** – manual `close()` in `finally` may miss closing on unexpected exceptions; risk of exhausted DB connections.
- **No transaction management** – multi‑statement operations (e.g., main + attribute inserts) are not atomic; partial writes possible on failure.
- **Hard‑coded IMSI loop** – assumes exactly 10 IMSI slots; mismatched list size could cause `IndexOutOfBounds` or silent data loss.
- **SQL injection safety** – reliance on external SQL strings; malformed entries in `MOVEDAO` could break statements.
- **Error handling granularity** – all exceptions are wrapped as generic `DatabaseException`; original exception type lost for callers.

# Usage
```java
// Example: insert a successful notification record
RawTableDataAccess dao = new RawTableDataAccess();
NotificationRecord rec = new NotificationRecord();
// populate rec fields...
try {
    dao.insertInRawTable(rec);
} catch (DatabaseException e) {
    // handle or retry
}
```
For debugging, enable Log4j DEBUG level for `com.tcl.move.dao.RawTableDataAccess` to view bound parameters and stack traces.

# Configuration
- **System properties** (read by `JDBCConnection`):  
  - `oracle.jdbc.url`, `oracle.jdbc.user`, `oracle.jdbc.password`  
  - `hive.jdbc.url`, `hive.jdbc.user`, `hive.jdbc.password`
- **Resource bundle**: `MOVEDAO.properties` located on the classpath; must contain keys: `insert.raw`, `insert.raw.reject`, `insert.location.raw`, `insert.location`, `insert.location.attr`, `insert.ocs`, `insert.esim`, `insert.simimei`.
- **Log4j configuration**: ensure logger `com.tcl.move.dao.RawTableDataAccess` is defined.

# Improvements
1. **Adopt try‑with‑resources** for `Connection` and `PreparedStatement` to guarantee closure and simplify code.
2. **Introduce transaction boundaries** for multi‑statement methods (e.g., `insertInLocationRawTable(LocationUpdateRecord)`) using `connection.setAutoCommit(false)` and commit/rollback logic to ensure atomicity.