# Summary
`DBDataException` is a custom checked exception that signals data‑related problems encountered while accessing or processing records in the MOVE‑Mediation‑Ingest database. It propagates a descriptive error message up the call stack, enabling the ingestion pipeline to differentiate data integrity failures from connectivity or generic runtime errors.

# Key Components
- **Class `DBDataException` extends `Exception`** – base for all data‑validation‑or‑corruption errors in the DB layer.  
- **`serialVersionUID`** – fixed UID (`6228590916239837910L`) for serialization compatibility.  
- **Constructor `DBDataException(String msg)`** – creates the exception with a human‑readable message.

# Data Flow
- **Input:** String error message supplied by the caller (DAO, service, or validator).  
- **Output:** Thrown `DBDataException` instance; no return value.  
- **Side Effects:** Triggers exception handling mechanisms; may be logged or cause transaction rollback.  
- **External Interactions:** None directly; used by components that interact with the relational database (JDBC, JPA, MyBatis, etc.).

# Integrations
- **DAO Layer:** Thrown when a query returns malformed data, violates constraints, or fails sanity checks.  
- **Business Logic:** Caught to apply compensation, alerting, or to skip malformed records.  
- **Reporting Modules:** May translate the exception into error metrics for the KLMReport subsystem.  
- **Other Exceptions:** Coexists with `DatabaseException`, `DBConnectionException`; higher‑level handlers often differentiate based on exception type.

# Operational Risks
- **Uncaught Propagation:** If not caught, may abort batch jobs, leading to incomplete ingestion. *Mitigation:* Centralized exception handler in the ingestion orchestrator.  
- **Loss of Context:** Only a message is stored; stack trace may lack root‑cause details. *Mitigation:* Wrap underlying exceptions using `Throwable cause` (future improvement).  
- **Serialization Mismatch:** Changing class structure without updating `serialVersionUID` could break distributed processing. *Mitigation:* Keep UID constant or manage versioning explicitly.

# Usage
```java
// Example in a DAO method
public Record fetchRecord(String id) throws DBDataException {
    try {
        // DB query logic...
        if (record == null) {
            throw new DBDataException("Record not found for ID: " + id);
        }
        return record;
    } catch (SQLException e) {
        // Convert to data‑specific exception
        throw new DBDataException("Database error while fetching ID " + id + ": " + e.getMessage());
    }
}
```
*Debug:* Set a breakpoint on the constructor or catch block to inspect the message.

# Configuration
- No environment variables or external configuration files are referenced by this class.

# Improvements
1. **Add cause chaining constructor** – `DBDataException(String msg, Throwable cause)` to preserve original stack trace.  
2. **Introduce error codes** – enum or int field to categorize data errors (e.g., `INVALID_FORMAT`, `MISSING_FIELD`).