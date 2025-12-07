# Summary
`JSONParseUtils` provides a stateless utility method `parseBalanceJSON` that converts a JSON‑encoded balance notification into a `BalanceRecord` DTO. It logs the raw payload, extracts required fields (with optional handling), populates the DTO, and wraps any parsing failures in a custom checked `ParseException`. The utility is used by `NotificationService` (and indirectly by the Kafka consumer) to transform inbound messages for persistence and downstream processing.

# Key Components
- **class `JSONParseUtils`**
  - `private static final Logger logger` – Log4j logger for traceability.
  - `public static BalanceRecord parseBalanceJSON(String balanceJSON) throws ParseException` – Core parser; builds a `BalanceRecord` from the supplied JSON string.
  - `private static String getStackTrace(Exception exception)` – Helper that returns a full stack trace as a string for logging.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | Raw JSON string (balance notification) | `new JSONObject(balanceJSON)` → field extraction → DTO population | `BalanceRecord` instance | Log raw JSON (`logger.info`) |
| 2 | Parsing errors (`JSONException` or generic `Exception`) | Capture exception → log stack trace (`logger.error`) → throw `ParseException` | Propagated exception to caller | None beyond logging |

External interactions: none (pure in‑memory transformation).  

# Integrations
- **`NotificationService.processBalanceRequest(String json)`** – Calls `JSONParseUtils.parseBalanceJSON` to obtain a `BalanceRecord` before persisting via `RawTableDataAccess`.
- **`NotificationConsumer`** – Retrieves Kafka messages, passes the payload string to `NotificationService`, which in turn uses this utility.
- **`MailService`** – Not directly used here; invoked only when `ParseException` propagates up to `NotificationService`.

# Operational Risks
- **Malformed JSON** – Triggers `ParseException`; downstream processing halts for that record. *Mitigation*: Dead‑letter queue or retry logic in `NotificationService`.
- **Missing mandatory fields** – `JSONObject.getString` throws if a required key is absent, causing a parse failure. *Mitigation*: Validate schema upstream or enhance parser with `optString` defaults.
- **Performance overhead of full stack trace logging** – May increase I/O under high error volume. *Mitigation*: Log only exception message at INFO level; reserve stack trace for DEBUG.

# Usage
```java
// Unit test / debugging example
String payload = "..."; // JSON string from Kafka or batch file
try {
    BalanceRecord rec = JSONParseUtils.parseBalanceJSON(payload);
    System.out.println(rec);
} catch (ParseException e) {
    e.printStackTrace(); // handle according to business rules
}
```
Run within the application context; no external services required.

# configuration
No environment variables or external config files are referenced by this class. It relies solely on Log4j configuration (log level, appenders) defined elsewhere in the application.

# Improvements
1. **Schema Validation** – Integrate JSON schema validation (e.g., using `org.everit.json.schema`) to produce clearer error messages for missing/invalid fields before DTO mapping.
2. **Optional Field Handling** – Replace `getString` with `optString` (and `optDouble`) for non‑critical fields to avoid unnecessary `ParseException` for tolerable omissions.