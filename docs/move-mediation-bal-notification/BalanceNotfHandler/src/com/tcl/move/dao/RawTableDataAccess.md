# Summary
`RawTableDataAccess` provides a DAO method `insertInBalanceRawTable` that persists a single `BalanceRecord` into the Hive table `mnaas.api_balance_update` using a prepared INSERT statement defined in `MOVEDAO.properties`. It obtains a Hive JDBC connection via `JDBCConnection`, logs activity, and translates any failure into a `DatabaseException` with the error category `Constants.ERROR_RAW`.

# Key Components
- **Class `RawTableDataAccess`**
  - `logger` – Log4j logger for operational tracing.
  - `daoBundle` – `ResourceBundle` loading `MOVEDAO.properties`.
  - `jdbcConn` – Instance of `JDBCConnection` used to acquire Hive connections.
  - **Method `insertInBalanceRawTable(BalanceRecord)`**
    - Retrieves SQL from `daoBundle`.
    - Acquires Hive `Connection`.
    - Prepares statement, binds 16 parameters from `BalanceRecord`.
    - Executes `executeUpdate()`.
    - Handles and logs exceptions, wrapping them in `DatabaseException`.
    - Closes `PreparedStatement` and `Connection` in `finally`.
  - **Private helper `getStackTrace(Exception)`**
    - Converts exception stack trace to a `String` for logging.

# Data Flow
| Stage | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| 1 | `BalanceRecord` instance (populated by upstream processing) | Extract fields via getters | N/A |
| 2 | `MOVEDAO.properties` (`insert.balance` SQL) | Load via `ResourceBundle` | Parameterised INSERT string |
| 3 | Hive JDBC connection (via `JDBCConnection.getConnectionHive()`) | Prepare statement, bind 16 values | PreparedStatement |
| 4 | Execution of `statement.executeUpdate()` | Insert row into Hive table `mnaas.api_balance_update` | Row persisted |
| 5 | Exceptions | Log stack trace, throw `DatabaseException` with `Constants.ERROR_RAW` | Failure propagation |
| 6 | Finally block | Close `PreparedStatement` and `Connection` | Resource cleanup |

External services:
- Hive/Impala cluster (JDBC endpoint).
- Logging infrastructure (Log4j).

# Integrations
- **Upstream**: Called by batch job components that transform raw balance‑update events into `BalanceRecord` objects.
- **Downstream**: Data becomes available to downstream analytics or reporting pipelines that query `mnaas.api_balance_update`.
- **Configuration**: Relies on `MOVEDAO.properties` for the INSERT statement and on `JDBCConnection` for Hive connection parameters (driver class, URL, credentials) supplied via system properties.
- **Error handling**: Propagates `DatabaseException` to the caller, which is expected to be caught by the batch framework for retry or alerting.

# Operational Risks
- **Resource leakage**: Failure to close `Connection`/`PreparedStatement` on unexpected exceptions could exhaust DB connections. *Mitigation*: Use try‑with‑resources (Java 7+) or ensure finally block never re‑throws.
- **SQL injection**: Not applicable due to prepared statement, but malformed `BalanceRecord` values could cause type errors. *Mitigation*: Validate `BalanceRecord` before DAO call.
- **Hive latency**: Insert may be slow under high volume, causing batch back‑pressure. *Mitigation*: Batch inserts or use Hive’s `INSERT … VALUES` multi‑row syntax.
- **Configuration drift**: Mismatch between `MOVEDAO.properties` placeholders and `BalanceRecord` fields leads to `SQLException`. *Mitigation*: Unit test property‑to‑field mapping.

# Usage
```java
// Example unit‑test snippet
BalanceRecord rec = new BalanceRecord();
// populate rec fields ...
RawTableDataAccess dao = new RawTableDataAccess();
try {
    dao.insertInBalanceRawTable(rec);
    System.out.println("Insert successful");
} catch (DatabaseException e) {
    e.printStackTrace();
}
```
Run within the batch job’s JVM with required system properties for Hive connection (e.g., `-Dhive.url=jdbc:hive2://host:10000/default -Dhive.user=... -Dhive.password=...`).

# configuration
- **System properties** (read by `JDBCConnection`):
  - `hive.driver` – Hive JDBC driver class.
  - `hive.url` – JDBC URL.
  - `hive.user` / `hive.password` – Credentials.
- **Config files**:
  - `MOVEDAO.properties` (on classpath) containing key `insert.balance` with 16 `?` placeholders.
- **Log4j configuration** – `log4j.properties` defining logger `com.tcl.move.dao.RawTableDataAccess`.

# Improvements
1. **Refactor resource handling** – Replace manual `close()` calls with try‑with‑resources to guarantee deterministic cleanup and eliminate the need to re‑throw in the finally block.
2. **Batch insertion support** – Add a method that accepts a `List<BalanceRecord>` and uses `addBatch()` / `executeBatch()` to improve throughput and reduce Hive round‑trips.