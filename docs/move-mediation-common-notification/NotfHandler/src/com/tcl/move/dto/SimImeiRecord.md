# Summary
`SimImeiRecord` is a plain‑old‑Java‑Object (POJO) that models a SIM‑IMEI change event received as JSON by the MOVE mediation notification handler. It stores all raw attributes extracted from the inbound payload, provides standard getters/setters, a helper to obtain the date portion of `eventDateTime`, and a `toString()` for logging. Instances are handed to DAO components for persistence into Hive/Oracle tables.

# Key Components
- **Class `SimImeiRecord`**
  - Private `String` fields: initiatorId, transactionId, sourceId, tenancyId, eventId, parentEventId, entityType, entityId, eventClass, eventDateTime, accountId, accountName, oldImei, newImei, sim, imsi, msisdn, oldTimestamp, newTimestamp, result, source, failReason.
  - Public getters/setters for each field.
  - `String getEventDate()` – returns the first 10 characters of `eventDateTime` (YYYY‑MM‑DD) or empty string.
  - Overridden `toString()` – concatenates all fields into a readable log format.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Inbound API | JSON payload from external telecom API (SIM‑IMEI change) | JSON deserializer (e.g., Jackson) maps keys to `SimImeiRecord` fields via setters | Populated `SimImeiRecord` instance |
| 2. Business Logic | `SimImeiRecord` object | Validation / enrichment (outside this class) | May modify fields via setters |
| 3. Persistence | `SimImeiRecord` | DAO layer converts fields to Hive/Oracle column values | Rows inserted/updated in target tables |
| 4. Logging / Monitoring | `SimImeiRecord` | `toString()` invoked | Human‑readable log entry |

No direct external service calls or queue interactions occur within this class.

# Integrations
- **Notification Handler** – parses inbound JSON and creates `SimImeiRecord`.
- **DAO Layer** – receives the POJO for persistence (`SimImeiRecordDao` or generic `NotificationDao`).
- **Logging Framework** – uses `toString()` for audit logs.
- **Potential Utility Classes** – date helpers may call `getEventDate()`.

# Operational Risks
- **NullPointerException** if `eventDateTime` is shorter than 10 characters; mitigated by length check before `substring`.
- **Data Loss** due to all fields being `String`; numeric or timestamp semantics are not enforced.
- **Inconsistent Logging** if any field contains commas or braces; consider escaping or structured logging.
- **Schema Drift** – adding/removing JSON fields requires POJO updates; lack of versioning may cause silent data loss.

# Usage
```java
// Example unit‑test snippet
SimImeiRecord rec = new SimImeiRecord();
rec.setInitiatorId("user123");
rec.setTransactionId("tx456");
rec.setEventDateTime("2024-11-30T14:22:01Z");
rec.setOldImei("123456789012345");
rec.setNewImei("543210987654321");

// Debug output
System.out.println(rec);               // invokes toString()
System.out.println(rec.getEventDate()); // prints "2024-11-30"
```
In production, the object is instantiated by the JSON deserializer within the notification handler.

# Configuration
The class itself does not reference environment variables or external configuration files. Runtime configuration (DB connections, queue endpoints, etc.) is handled by the surrounding DAO and handler components.

# Improvements
1. **Add Input Validation** – implement a `validate()` method that checks mandatory fields, length of `eventDateTime`, and numeric format of IMEI values; throw a domain‑specific exception on failure.  
2. **Replace Raw Strings with Typed Fields** – use `java.time.OffsetDateTime` for `eventDateTime` and a dedicated `IMEI` value object to enforce format and enable safer date handling.