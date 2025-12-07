# Summary
`DatabaseException` is a custom checked exception used throughout the Move‑Mediation add‑on notification service to encapsulate database‑related error conditions. It carries an optional error‑type identifier to aid downstream error handling and logging.

# Key Components
- **class `DatabaseException` extends `Exception`**
  - `private String errorType` – optional categorisation of the database error.
  - Constructors  
    - `DatabaseException(String msg, String errorType)` – sets message and error type.  
    - `DatabaseException(String msg)` – sets only the message.
  - Accessors  
    - `String getErrorType()` – returns the error‑type.  
    - `void setErrorType(String errorType)` – mutates the error‑type.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| Input   | Method parameters (`msg`, `errorType`) | Message and optional classification supplied by DAO or service layers when a JDBC operation fails. |
| Output  | Thrown exception | Propagates up the call stack to be caught by higher‑level handlers (e.g., service façade, batch runner). |
| Side‑effects | Logging (via callers) | Callers typically log the exception details before re‑throwing or handling. |
| External services | None directly | The exception itself does not contact external systems; it is a carrier for error context. |

# Integrations
- **`JDBCConnection`** – wraps driver loading and connection acquisition; on failure throws `DatabaseException`.
- **`RawTableDataAccess`** – catches `SQLException`, wraps it in `DatabaseException` and propagates.
- **Higher‑level services / batch jobs** – catch `DatabaseException` to trigger retry, alerting, or dead‑letter handling.

# Operational Risks
- **Unchecked propagation** – If callers forget to catch `DatabaseException`, the thread may terminate, causing batch loss. *Mitigation*: enforce catch blocks in all DAO entry points; add unit tests verifying handling.
- **Loss of root cause** – Original `SQLException` stack trace may be discarded if only the message is passed. *Mitigation*: include the original exception as a cause (`new DatabaseException(msg, errorType, cause)`) or store it in a field.
- **Inconsistent errorType usage** – Varying strings can hinder automated monitoring. *Mitigation*: define an enum of canonical error types and enforce usage.

# Usage
```java
try {
    Connection con = jdbcConnection.getHiveConnection();
    // perform DB work
} catch (SQLException e) {
    throw new DatabaseException("Failed to obtain Hive connection", "HIVE_CONN_ERROR");
}
```
Debugging: set a breakpoint on the `DatabaseException` constructors; inspect `msg` and `errorType`.

# configuration
No environment variables or external configuration are read by this class. It relies on callers to supply context strings.

# Improvements
1. **Add cause chaining** – Provide a constructor `DatabaseException(String msg, String errorType, Throwable cause)` and store the cause via `super(msg, cause)`.
2. **Standardise error types** – Introduce an `enum DatabaseErrorType` and replace the mutable `String errorType` with a typed field to enforce consistency.