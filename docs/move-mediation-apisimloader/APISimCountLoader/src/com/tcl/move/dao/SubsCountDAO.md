# Summary
`SubsCountDAO` implements data‑access operations for the APISimCountLoader component. It retrieves partition dates, truncates and populates monthly and yearly SIM count tables in the `mnaas` Hive schema, and manages yearly partition lifecycle. All operations are executed via Impala/Hive JDBC connections.

# Key Components
- **class `SubsCountDAO`**
  - `logger` – Log4j logger.
  - `daoBundle` – `ResourceBundle` loading `MOVEDAO.properties`.
  - `jdbcConn` – Helper to obtain Hive JDBC connections.
- **public methods**
  - `String getLastPartFromRaw()` – Returns latest `partition_date` from `move_sim_inventory_status`.
  - `String getLastPartFromAggr()` – Returns latest `partition_date` from `move_sim_inventory_count`.
  - `String getMonthPartDate()` – Returns latest `status_date` from `move_curr_month_count`.
  - `void loadMonthTable()` – Truncates `move_curr_month_count` and inserts active/suspended counts and activity counts for the current (or previous) month.
  - `void loadYearTable()` – Drops current month‑year partition, inserts active counts and activity counts for the year, and prunes partitions older than 12 months.
- **private static `String getStackTrace(Exception)`** – Utility to convert exception stack trace to a string for logging.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | SQL strings from `MOVEDAO.properties` | Execute via JDBC `Statement`/`PreparedStatement` | Retrieved partition dates (`String`). |
| 2 | Current date utilities from `SubsCountUtils` | Build date ranges for month/year processing | Date strings (`monthStart`, `monthDates`, etc.). |
| 3 | `loadMonthTable` | Truncate `move_curr_month_count`; insert active SIM aggregates; loop over daily activity dates and insert activity counts. | Updated `move_curr_month_count` partitions. |
| 4 | `loadYearTable` | Drop existing month‑year partition; insert active counts for the month; insert activity counts; query partition count; if >12, drop oldest partition. | Updated `move_curr_year_count` partitions; possible partition cleanup. |
| 5 | Exceptions | Captured, logged, re‑thrown as `DatabaseException`. | Propagation of error state to caller. |

External services:
- Hive/Impala cluster accessed via JDBC (connection details from system properties).
- `SubsCountUtils` for date calculations (no external I/O).

# Integrations
- **`JDBCConnection`** – Provides Hive JDBC connection; used for all DB interactions.
- **`MOVEDAO.properties`** – Supplies all SQL statements referenced by DAO methods.
- **`SubsCountUtils`** – Supplies date logic (first‑of‑month detection, month start/end dates, list of dates).
- **`APISimCountLoader`** (not shown) – Likely orchestrates DAO calls to drive the ETL workflow.
- **Log4j** – Centralized logging for operational monitoring.

# Operational Risks
- **Resource leakage** – `ResultSet`, `Statement`, and `Connection` close order is inconsistent; may leave open resources on exception. *Mitigation*: Use try‑with‑resources.
- **Hard‑coded SQL in properties** – Changes to schema require property updates; no compile‑time validation. *Mitigation*: Add integration tests for each query.
- **Date handling edge cases** – Reliance on `SubsCountUtils` for month boundaries; incorrect timezone could cause partition misalignment. *Mitigation*: Ensure UTC consistency and unit‑test date utilities.
- **Partition pruning logic** – Fixed limit of 12 partitions; if business rule changes, code must be updated. *Mitigation*: Externalize partition retention policy.

# Usage
```java
// Example from a unit test or main method
SubsCountDAO dao = new SubsCountDAO();
try {
    String rawPart = dao.getLastPartFromRaw();
    System.out.println("Raw partition: " + rawPart);

    dao.loadMonthTable();   // Populate monthly counts
    dao.loadYearTable();    // Populate yearly counts
} catch (DatabaseException e) {
    e.printStackTrace();
}
```
For debugging, set Log4j level to `DEBUG` in `log4j.properties` and inspect generated Hive queries in the logs.

# configuration
- **Environment variables / system properties** required by `JDBCConnection` (e.g., `impala.host`, `impala.port`, `impala.database`).
- **`MOVEDAO.properties`** – Must be on the classpath; contains keys:
  - `part.date.raw`, `part.date.aggr`, `part.date.month`
  - `truncate.month`, `insert.month.active`, `insert.month.activity`, `insert.lastmonth.active`
  - `drop.part.year`, `insert.year.active`, `insert.year.activity`
  - `get.partiton.count`, `get.min.part`
- **Log4j configuration** – `log4j.properties` defines root logger and appenders.

# Improvements
1. Refactor all JDBC resource handling to Java 7+ try‑with‑resources to guarantee deterministic closure and reduce boilerplate.
2. Externalize partition retention limit and date‑format patterns to a dedicated configuration file; inject via constructor for easier testing and runtime tuning.