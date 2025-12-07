# Summary
`LocationUpdateRecord` is a POJO that represents a location‑update event received from external APIs in the MOVE mediation notification handler. It encapsulates all fields extracted from the inbound JSON payload, provides getters/setters, a few convenience string‑extraction helpers, and a map for arbitrary custom attributes. Instances are passed to DAO classes for persistence into Hive/Oracle tables.

# Key Components
- **Class `LocationUpdateRecord`**
  - Private String fields for core event data (initiator, transaction, entity, timestamps, device identifiers, network element details, etc.).
  - `Map<String,String> customAttributes` for extensible key/value pairs.
  - Standard JavaBean getters and setters for each field.
  - `getStrEventDate()` – returns the date portion (`yyyy‑MM‑dd`) of `eventDateTime`.
  - `getStrImeiDate()` – alias for raw `imeiDate`.
  - `addCustomAttribute(String key, String value)` – inserts into `customAttributes`.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. API inbound | JSON payload from external location‑update API | JSON deserializer (e.g., Jackson) populates a `LocationUpdateRecord` instance via setters | In‑memory POJO |
| 2. Business logic | `LocationUpdateRecord` object | Validation, enrichment, optional custom attribute addition | Enriched POJO |
| 3. Persistence | Enriched POJO | DAO (`RawTableDataAccess` or similar) extracts fields and writes to Hive/Oracle tables | DB rows; no direct side‑effects from this class |
| 4. Logging / monitoring | POJO `toString()` (inherited from `Object`) used by callers | Serialized for audit logs | Log entries |

# Integrations
- **JSON deserialization layer** – typically Jackson/Gson mapping JSON keys to POJO setters.
- **DAO layer** – `RawTableDataAccess` or other persistence components read the POJO fields to construct INSERT statements for Hive/Oracle.
- **Custom attribute handling** – downstream components may iterate `customAttributes` to store dynamic columns or forward to downstream queues (e.g., Kafka).
- **Logging framework** – callers log the POJO for traceability.

# Operational Risks
- **Missing/invalid fields** – No validation in POJO; malformed JSON can produce null fields leading to DB constraint violations. *Mitigation*: add validation layer before DAO call.
- **Date parsing assumptions** – `getStrEventDate()` blindly substrings first 10 characters; malformed timestamps cause `StringIndexOutOfBoundsException`. *Mitigation*: guard with length check or use proper date parsing.
- **Custom attribute map concurrency** – Not thread‑safe if the same instance is shared across threads. *Mitigation*: ensure each thread works on its own POJO or wrap map with `Collections.synchronizedMap`.
- **Schema drift** – Adding new fields requires code changes; no versioning. *Mitigation*: use a flexible schema (e.g., Avro) for downstream storage.

# Usage
```java
// Example: deserialize JSON and persist
ObjectMapper mapper = new ObjectMapper();
LocationUpdateRecord rec = mapper.readValue(jsonString, LocationUpdateRecord.class);

// Optional enrichment
rec.addCustomAttribute("sourceSystem", "MNO_X");

// Persist
rawTableDao.insertLocationUpdate(rec);
```
*Debug*: Set a breakpoint in any getter/setter or in `addCustomAttribute` to inspect field values after deserialization.

# Configuration
- No environment variables or config files are referenced directly by this class.
- Dependent components (JSON mapper, DAO, logging) rely on external configuration (e.g., `application.properties`, Spring context).

# Improvements
1. **Add field validation** – Implement a `validate()` method that checks mandatory fields, date formats, and numeric constraints; throw a domain‑specific exception on failure.
2. **Replace manual date substring with robust parsing** – Use `java.time.LocalDateTime` to parse `eventDateTime` and expose `getEventDate()` returning a `LocalDate`. This prevents `StringIndexOutOfBoundsException` and improves type safety.