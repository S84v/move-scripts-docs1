# Summary
`ParseException` is a custom checked exception used throughout the Move mediation balance‑notification batch to signal errors encountered while parsing JSON payloads. It encapsulates an error message and, optionally, the originating feed identifier and type, enabling downstream components to classify and handle parsing failures distinctly from other error conditions.

# Key Components
- **`com.tcl.move.exceptions.ParseException`**
  - Extends `java.lang.Exception`.
  - Serial version UID for serialization compatibility.
  - Constructor `ParseException(String msg, String feed, String type)` – captures a message; parameters `feed` and `type` are currently unused but reserved for future context enrichment.
  - Constructor `ParseException(String msg)` – captures a simple error message.

# Data Flow
- **Input:** Error context supplied by calling code (message, optional feed name, optional type).
- **Output:** Thrown exception object propagated up the call stack.
- **Side Effects:** None (exception carries data only).
- **External Interactions:** None directly; consumed by parsers handling JSON from Move API, and by higher‑level handlers that may log, alert, or trigger retries.

# Integrations
- Invoked by JSON parsing utilities within the `com.tcl.move` package (e.g., DTO deserializers, API response handlers).
- Caught by service layers or batch jobs that orchestrate balance‑notification processing; these layers may map the exception to `DatabaseException`, `MailException`, or other domain‑specific error handling pathways.

# Operational Risks
- **Unused parameters (`feed`, `type`)** – may cause confusion; risk of lost diagnostic information if not stored.
  - *Mitigation:* Populate fields or remove parameters.
- **Unchecked propagation** – if not caught, may abort batch jobs.
  - *Mitigation:* Ensure all parsing entry points declare or handle `ParseException`.
- **Serialization UID mismatch** – unlikely but could affect distributed exception handling.
  - *Mitigation:* Keep UID stable; document versioning policy.

# Usage
```java
try {
    // Example JSON parsing block
    JSONObject obj = new JSONObject(jsonString);
    // parsing logic...
} catch (JSONException e) {
    throw new ParseException("Failed to parse balance record", "BalanceFeed", "JSON");
}
```
Debugging: Set a breakpoint on the `ParseException` constructors to inspect supplied arguments.

# configuration
No environment variables or external configuration files are referenced by this class.

# Improvements
- **Store feed and type:** Add private fields for `feed` and `type`, expose getters, and pass values to `super(msg)` via a formatted message or a custom payload.
- **Remove unused parameters:** If contextual data is not required, delete the three‑argument constructor to simplify the API.