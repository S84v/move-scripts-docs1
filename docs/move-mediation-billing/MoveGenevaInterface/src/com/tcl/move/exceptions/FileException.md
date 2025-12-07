# Summary
`FileException` is a custom checked exception used in the Move‑Geneva mediation system to represent file‑related error conditions. It propagates descriptive error messages up the call stack, enabling uniform handling, logging, and transaction rollback in production.

# Key Components
- **class `FileException` extends `Exception`**
  - `serialVersionUID = 6228590916239837910L` – ensures serialization compatibility.
  - Constructor `FileException(String msg)` – forwards the supplied message to the superclass.

# Data Flow
- **Input:** String error message supplied by calling code when a file operation fails.
- **Output:** An instantiated `FileException` object thrown to the caller.
- **Side Effects:** None; exception carries only diagnostic information.
- **External Interactions:** May be caught by higher‑level services that log to monitoring systems or trigger compensating actions (e.g., DB rollback, message queue alerts).

# Integrations
- Propagated through DAO and service layers that perform file I/O (e.g., reading billing files, writing audit logs).
- Caught by global exception handlers in the Move‑Geneva application to produce standardized error responses and audit entries.
- Co‑exists with other custom exceptions (`DatabaseException`, `DBConnectionException`, `DBDataException`) for consistent error taxonomy.

# Operational Risks
- **Risk:** Uncaught `FileException` leads to thread termination and incomplete processing.
  - **Mitigation:** Ensure all file‑access code is wrapped in try‑catch blocks that handle `FileException` and invoke centralized error handling.
- **Risk:** Overly generic message hampers root‑cause analysis.
  - **Mitigation:** Include contextual data (file path, operation type) when constructing the exception.

# Usage
```java
try {
    // Example file operation
    Files.readAllLines(Paths.get("/opt/move/input.txt"));
} catch (IOException e) {
    throw new FileException("Failed to read input file: " + e.getMessage());
}
```
Debug by setting breakpoints on the `throw new FileException(...)` line or by inspecting stack traces in logs.

# configuration
No environment variables or external configuration files are referenced directly by `FileException`. It relies on the surrounding application’s logging and exception‑handling configuration.

# Improvements
- **TODO 1:** Add overloaded constructor accepting `Throwable cause` to preserve original stack trace.
- **TODO 2:** Implement a utility method to format standardized error messages (e.g., include file name, operation, timestamp).