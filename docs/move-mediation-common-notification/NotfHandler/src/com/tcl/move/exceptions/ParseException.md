# Summary
`ParseException` is a custom checked exception used in the MOVE mediation notification handler to signal errors encountered while parsing JSON payloads. It propagates an error message (and optionally feed and type information) to calling components for centralized error handling and logging.

# Key Components
- **Class `ParseException` (extends `Exception`)**
  - `serialVersionUID` – serialization identifier.
  - Constructor `ParseException(String msg, String feed, String type)` – creates exception with a message; `feed` and `type` parameters are currently unused.
  - Constructor `ParseException(String msg)` – creates exception with only a message.

# Data Flow
- **Input:** Error context (message, optional feed identifier, optional type) supplied by JSON parsing logic.
- **Output:** Thrown `ParseException` instance propagated up the call stack.
- **Side Effects:** None; exception handling may trigger logging or error‑response generation in higher layers.
- **External Services/DBs/Queues:** None directly; used by components that interact with external services (e.g., Hive, Oracle) indirectly via the mediation pipeline.

# Integrations
- Invoked by JSON parsing utilities within the `com.tcl.move` package (e.g., parsers handling notification payloads).
- Caught by higher‑level handlers such as `NotificationHandler`, `MoveMediator`, or generic exception interceptors that translate the exception into error responses or audit logs.
- May be wrapped or re‑thrown as `DatabaseException` or `MailException` depending on downstream processing.

# Operational Risks
- **Unused parameters (`feed`, `type`)** – may cause confusion; risk of missing contextual information in logs.
  - *Mitigation:* Populate superclass with detailed message or store parameters in fields for later retrieval.
- **Unchecked propagation** – if not caught, can cause thread termination and loss of processing continuity.
  - *Mitigation:* Ensure all parsing entry points have try‑catch blocks that handle `ParseException` and trigger appropriate fallback or alerting.

# Usage
```java
try {
    JsonNode node = objectMapper.readTree(jsonString);
    // parsing logic …
} catch (IOException e) {
    throw new ParseException("Failed to parse JSON payload", feedId, "JSON");
}
```
Debug by setting breakpoints on the constructors or catching the exception in unit tests.

# Configuration
No environment variables or external configuration files are referenced by this class.

# Improvements
- **Add fields for `feed` and `type`** and expose them via getters to retain context for logging and monitoring.
- **Override `toString()`** to include feed and type information, improving traceability in logs.