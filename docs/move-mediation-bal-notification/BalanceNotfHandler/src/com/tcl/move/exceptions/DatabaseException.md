# Summary
`DatabaseException` is a custom checked exception used throughout the Move mediation balance‑notification batch to encapsulate database‑related errors. It carries an optional error‑type identifier that downstream components (e.g., DAOs) use to classify failures for logging, alerting, and retry logic.

# Key Components
- **Class `com.tcl.move.exceptions.DatabaseException`**
  - Extends `java.lang.Exception`.
  - Field `private String errorType` – optional categorical identifier.
  - Constructors:
    - `DatabaseException(String msg, String errorType)` – sets message and category.
    - `DatabaseException(String msg)` – sets message only.
  - Accessors:
    - `String getErrorType()`
    - `void setErrorType(String errorType)`

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| Caller (DAO, service layer) | → `DatabaseException` | Throws when a JDBC operation fails (e.g., connection error, SQL exception). |
| `DatabaseException` | → Caller | Propagates error message and optional `errorType` up the call stack. |
| Logging framework (Log4j) | ← Caller | Captures exception details for audit. |
| No external I/O performed by the class itself. |

# Integrations
- **`JDBCConnection`** – catches `SQLException` and re‑throws as `DatabaseException`.
- **`RawTableDataAccess`** – catches generic `Exception`, wraps in `DatabaseException` with `Constants.ERROR_RAW`.
- Any component that declares `throws DatabaseException` in its method signature participates in the error‑propagation chain.

# Operational Risks
- **Unchecked propagation** – If callers do not handle `DatabaseException`, the batch may abort unexpectedly. *Mitigation*: enforce handling at service layer; implement global exception handler.
- **Loss of root cause** – Original `SQLException` stack trace may be truncated if not passed as cause. *Mitigation*: add constructor overload accepting `Throwable cause` and use `super(msg, cause)`.
- **Inconsistent errorType usage** – Missing or mismatched categories hinder monitoring. *Mitigation*: define an enum of error types and enforce usage.

# Usage
```java
try {
    Connection conn = JDBCConnection.getHiveConnection();
    // perform DB work
} catch (SQLException e) {
    throw new DatabaseException("Hive connection failed", Constants.ERROR_HIVE);
}
```
To debug, set Log4j level for the calling class to `DEBUG` and inspect the thrown `DatabaseException` message and `errorType`.

# configuration
- No environment variables or external config files are read directly by this class.
- Relies on callers to supply appropriate `errorType` strings (e.g., `Constants.ERROR_RAW`, `Constants.ERROR_HIVE`).

# Improvements
1. **Add cause‑preserving constructor**: `DatabaseException(String msg, String errorType, Throwable cause)` to retain original stack trace.
2. **Replace raw `String errorType` with a typed enum** (e.g., `enum DatabaseErrorType`) to enforce compile‑time consistency.