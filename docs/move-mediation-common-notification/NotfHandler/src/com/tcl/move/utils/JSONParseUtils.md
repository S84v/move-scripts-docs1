# Summary
`JSONParseUtils` provides a collection of static parsing utilities that convert inbound JSON payloads from various telecom move‑related events (SIM swap, MSISDN swap, status update, OTA, location update, subscription, IMEI change, OCS bypass, eSIM provisioning, bulk eSIM) into strongly‑typed DTO objects used downstream in the Move‑Mediation pipeline.

# Key Components
- **Class `JSONParseUtils`**
  - `parseSIMSwapJSON(String)` → `NotificationRecord`
  - `parseMSISDNSwapJSON(String)` → `NotificationRecord`
  - `parseStatusUpdateJSON(String)` → `NotificationRecord`
  - `parseOtaJSON(String)` → `NotificationRecord`
  - `parseLocationRawUpdateJSON(String)` → `LocationUpdateRawRecord`
  - `parseLocationUpdateJSON(String)` → `LocationUpdateRecord`
  - `parseSubsJSON(String)` → `NotificationRecord`
  - `parseSimImeiJSON(String)` → `SimImeiRecord`
  - `parseOCSBypassJSON(String)` → `OCSRecord`
  - `parseESimJSON(String)` → `ESimRecord`
  - `parseBulkESimJSON(String)` → `List<ESimRecord>`
  - `getStackTrace(Exception)` → `String` (private helper)

# Data Flow
| Method | Input | Processing | Output | Side Effects |
|--------|-------|------------|--------|--------------|
| `parse*JSON` | Raw JSON string received from external event source (REST API, message broker) | Instantiates `JSONObject`, extracts fields, populates DTO, logs parsed object | Returns DTO; logs at INFO level; on error logs stack trace and throws `ParseException` |
| `parseBulkESimJSON` | JSON containing a global transaction and an array of transaction objects | Loops over `transactions` array, creates `ESimRecord` per entry, aggregates into `List` | Returns list; logs each record and total count |
| `getStackTrace` | `Exception` | Captures stack trace via `StringWriter`/`PrintWriter` | Returns string for logging |

No external services, DB writes, or message queue interactions occur within this class; it is pure transformation.

# Integrations
- **DTO Packages**: `com.tcl.move.dto.*` – objects consumed by downstream processors (e.g., notification handlers, persistence services).
- **Constants**: `com.tcl.move.constants.Constants` – provides notification type identifiers.
- **Exception Handling**: `com.tcl.move.exceptions.ParseException` – propagated to callers for error handling.
- **Logging**: Apache Log4j – integrates with the application’s logging framework.
- **Caller Context**: Typically invoked by REST controllers or message listeners that receive the raw JSON payloads.

# Operational Risks
- **Schema Drift**: Missing or renamed JSON fields cause `JSONException` → `ParseException`. *Mitigation*: schema validation layer before parsing or tolerant parsing with default values.
- **Performance Overhead**: Repeated creation of `JSONObject` for large payloads may increase GC pressure. *Mitigation*: reuse parsers or switch to streaming JSON parser (Jackson `JsonParser`).
- **Logging Sensitive Data**: Full DTOs are logged at INFO, potentially exposing PII (MSISDN, ICCID). *Mitigation*: mask sensitive fields before logging or downgrade to DEBUG.
- **Null Handling**: Optional fields accessed via `has()` but some branches incorrectly check `jObject.has("iccidApplet")` instead of `appletObj`. May cause `JSONException`. *Mitigation*: correct field checks and add unit tests.

# Usage
```java
// Example in a unit test or service method
String payload = "..."; // JSON from external source
try {
    NotificationRecord rec = JSONParseUtils.parseSIMSwapJSON(payload);
    // Pass `rec` to downstream processor
} catch (ParseException e) {
    // Handle parsing failure (e.g., dead‑letter queue)
}
```
Debugging: set Log4j level for `com.tcl.move.utils.JSONParseUtils` to DEBUG to view raw JSON and parsed DTO.

# Configuration
- **Log4j Configuration**: `log4j.properties` or XML defines logger `com.tcl.move.utils.JSONParseUtils`.
- No environment variables or external config files are referenced directly in this class.

# Improvements
1. **Replace org.json with Jackson** – provides schema validation, streaming parsing, and better performance.
2. **Introduce a generic parsing framework** – reduce code duplication by mapping JSON fields to DTOs via reflection or a configuration map, and centralize default‑value handling.