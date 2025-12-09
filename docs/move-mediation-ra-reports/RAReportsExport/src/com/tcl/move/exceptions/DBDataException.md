# Summary
`DBDataException` is a custom checked exception used throughout the Move‑Mediation revenue‑assurance services to signal data‑related errors encountered when interacting with the database. Throwing this exception enables centralized error handling, logging, and transaction rollback for data integrity issues.

# Key Components
- **class `DBDataException` extends `Exception`**
  - `private static final long serialVersionUID = 6228590916239837910L;` – ensures serialization compatibility.
  - `public DBDataException(String msg)` – constructs the exception with a descriptive error message.

# Data Flow
- **Input:** String error message supplied by calling code when a data validation or integrity problem is detected.
- **Output:** Propagation of a `DBDataException` up the call stack.
- **Side Effects:** Triggers any surrounding `catch (DBDataException e)` blocks to log, rollback transactions, or alert monitoring systems.
- **External Services/DBs:** None directly; the exception is raised in response to failures from database access layers (e.g., DAO, repository classes).

# Integrations
- Referenced by DAO/repository implementations in `com.tcl.move.dao` and service classes in `com.tcl.move.service`.
- Caught alongside `DatabaseException` and `DBConnectionException` in higher‑level error‑handling utilities (e.g., `ExceptionHandler`, Spring `@ControllerAdvice`).
- May be mapped to HTTP error responses (e.g., 500) in REST controllers.

# Operational Risks
- **Risk:** Uncaught `DBDataException` leads to application crash or incomplete transaction rollback.  
  **Mitigation:** Ensure all data‑access entry points declare or handle the exception; integrate with global exception handler.
- **Risk:** Over‑use for non‑critical validation may obscure root cause.  
  **Mitigation:** Reserve for genuine data integrity violations; use specific validation exceptions for input errors.
- **Risk:** Missing serialVersionUID updates after class changes can break serialization compatibility.  
  **Mitigation:** Maintain versioning discipline; regenerate UID when class structure changes.

# Usage
```java
// Example in a DAO method
public void insertRecord(Record rec) throws DBDataException, DBConnectionException {
    try {
        // validation logic
        if (rec.getId() == null) {
            throw new DBDataException("Record ID cannot be null");
        }
        // JDBC insert...
    } catch (SQLException e) {
        throw new DBConnectionException("Unable to connect to DB", e);
    }
}

// Debugging
// 1. Set breakpoint on DBDataException constructor.
// 2. Run unit test or invoke service method.
// 3. Verify that the exception message propagates to the global handler.
```

# Configuration
- No environment variables or external configuration files are referenced directly by this class.
- Relies on application‑wide logging and exception‑handling configurations (e.g., `logback.xml`, Spring `@ControllerAdvice`).

# Improvements
1. **Add cause chaining constructor** – `public DBDataException(String msg, Throwable cause)` to preserve original stack trace.
2. **Document intended usage** – Javadoc tags describing when to throw vs. when to use other validation exceptions.