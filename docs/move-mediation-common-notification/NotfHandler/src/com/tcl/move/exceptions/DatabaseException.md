# Summary
`DatabaseException` is a custom checked exception used throughout the Move Mediation Notification Handler (MNAAS) to signal errors originating from database operations (Hive/Oracle persistence). It carries an optional `errorType` identifier for downstream error handling and logging.

# Key Components
- **Class `DatabaseException` (extends `Exception`)**
  - `String errorType` – optional categorisation of the database error.
  - Constructors:
    - `DatabaseException(String msg, String errorType)` – sets message and error type.
    - `DatabaseException(String msg)` – sets only the message.
  - Accessors:
    - `String getErrorType()`
    - `void setErrorType(String errorType)`

# Data Flow
- **Input:** Exception is instantiated with an error message (and optionally an error type) by DAO or service layers when a database operation fails.
- **Output:** Propagates up the call stack as a checked exception; caught by higher‑level handlers that may log, transform, or trigger retry/alert mechanisms.
- **Side Effects:** None intrinsic; side effects arise from catch blocks (e.g., logging, metrics, transaction rollback).
- **External Services/DBs:** Hive, Oracle, or any relational/NoSQL store accessed via DAO components that throw this exception.

# Integrations
- Referenced by DAO implementations in `move-mediation-common-notification\NotfHandler\src\com\tcl\move\dao\*`.
- Caught in service layer classes (e.g., `*Service.java`) and in the main notification handler to translate into HTTP error responses or to push to monitoring queues (e.g., Kafka, SNS).

# Operational Risks
- **Unchecked propagation:** If not caught, the exception can abort processing pipelines, causing data loss.
  - *Mitigation:* Ensure all DAO calls are wrapped in try‑catch blocks that handle `DatabaseException`.
- **Loss of error context:** Only `errorType` is stored; stack trace may be insufficient for root‑cause analysis.
  - *Mitigation:* Include original exception as cause (`new DatabaseException(msg, errorType, cause)`) in future revisions.
- **Serialization issues:** As a checked exception, it may be transmitted across JVM boundaries without proper handling.
  - *Mitigation:* Convert to a serialisable error DTO before sending to external systems.

# Usage
```java
try {
    dao.save(record);
} catch (DatabaseException de) {
    logger.error("DB error [{}]: {}", de.getErrorType(), de.getMessage());
    // optional: retry, alert, or rethrow as runtime
}
```
Debug by setting a breakpoint on the constructors or by enabling logging of the exception message and `errorType`.

# Configuration
No environment variables or external config files are directly referenced by this class. Configuration for database connections resides elsewhere (e.g., `application.properties`, Hive/Oracle datasource configs).

# Improvements
1. Add a constructor that accepts a `Throwable cause` to preserve original stack traces:
   ```java
   public DatabaseException(String msg, String errorType, Throwable cause) {
       super(msg, cause);
       this.errorType = errorType;
   }
   ```
2. Implement `Serializable` explicitly and define a stable `serialVersionUID` if the exception may be transmitted across JVMs.