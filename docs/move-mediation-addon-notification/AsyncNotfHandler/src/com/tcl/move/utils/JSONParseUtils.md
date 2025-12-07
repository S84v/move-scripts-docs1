# Summary
`JSONParseUtils` provides a single, production‑critical function that converts the raw asynchronous notification JSON payload received from the Move‑Mediation add‑on Kafka topic into a strongly‑typed `AddonRecord` DTO. It validates limited structural constraints, logs parsing activity, and translates any JSON‑related failures into the custom checked `ParseException` for upstream handling.

# Key Components
- **class `JSONParseUtils`**
  - `private static final Logger logger` – Log4j logger for traceability.
  - `public static AddonRecord parseAsyncJSON(String asyncJSON) throws ParseException` – Core parser; builds an `AddonRecord` from the JSON string.
  - `private static String getStackTrace(Exception e)` – Utility to convert an exception stack trace to a string for logging.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | `asyncJSON` (String) – raw payload from Kafka | `new JSONObject(asyncJSON)` → field extraction, optional checks, array handling | Populated `AddonRecord` instance |
| 2 | – | Logging of successful parse (`logger.info`) | Operational audit trail |
| 3 | – | On any `JSONException` or generic `Exception` | Log raw JSON + stack trace, throw `ParseException` (checked) |
| 4 | – | Validation of limited constraints (max 2 details, max 2 customAttributes) | `ParseException` if violated |

Side effects: writes to application logs; may expose PII in logs (full JSON).

# Integrations
- **`NotificationConsumer`** – Calls `JSONParseUtils.parseAsyncJSON` for each polled Kafka record; on `ParseException` triggers alert e‑mail via `MailService`.
- **`NotificationService`** – Receives the `AddonRecord` returned from the parser for persistence.
- **`AddonRecord` DTO** – Populated object consumed downstream (DB insert, further business logic).

# Operational Risks
- **Schema drift** – Adding/removing fields or changing nesting will cause `JSONException` and alert storms. *Mitigation*: versioned schema validation, unit tests against sample payloads.
- **Array length assumptions** – Fixed handling of `details` (max 1) and `customAttributes` (max 2) may reject legitimate messages. *Mitigation*: relax constraints or iterate over all elements.
- **PII leakage** – Full JSON is logged on error. *Mitigation*: mask sensitive fields (e.g., `msisdn`, `iccid`) before logging.
- **Unchecked `null` values** – Direct `getString` calls on mandatory fields will throw if missing. *Mitigation*: use `optString` with defaults or explicit presence checks.

# Usage
```java
String payload = /* fetch from Kafka or test fixture */;
try {
    AddonRecord rec = JSONParseUtils.parseAsyncJSON(payload);
    // proceed with business processing
} catch (ParseException e) {
    // handle parsing failure (e.g., alert, dead‑letter)
}
```
For debugging, enable Log4j at `DEBUG` level for `com.tcl.move.utils.JSONParseUtils`.

# Configuration
- No environment variables or external config files are referenced directly.
- Relies on Log4j configuration (`log4j.properties` / `log4j.xml`) for logger behavior.

# Improvements
1. **Schema‑driven validation** – Integrate a JSON Schema validator (e.g., everit‑json‑schema) to enforce structure and produce detailed error messages instead of ad‑hoc length checks.
2. **Robust optional handling** – Replace direct `getString`/`getInt` calls with `optString`, `optInt`, and iterate over `details`/`customAttributes` collections to support future extensions without throwing `ParseException`.