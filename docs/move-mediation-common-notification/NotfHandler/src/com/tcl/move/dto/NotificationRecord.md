# Summary
`NotificationRecord` is a POJO that represents a generic notification payload received from external APIs in the MOVE mediation system. It is populated from JSON by the notification handler, carried through business logic, and persisted by DAO components to Hive/Oracle tables.

# Key Components
- **Class `NotificationRecord`**
  - Private fields for core notification attributes (ids, timestamps, values, result, failure info, subscriber data, IMSIs list, etc.).
  - Standard getters/setters for each field.
  - `getStrEventDate()` – returns the date part (`yyyy‑MM‑dd`) of `eventDateTime`.
  - `addIMSI(String)` / `getIMSIs()` – manage a mutable list of IMSI strings.
  - `setSbsMember(Boolean)` – converts Boolean to `"Y"`/`"N"` string.
  - Overridden `toString()` – concise log‑friendly representation.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| JSON ingestion | Raw JSON from external API (e.g., eSIM provisioning, location update) | Deserialization (Jackson/Gson) → `NotificationRecord` instance | In‑memory object |
| Business handling | `NotificationRecord` | Validation, enrichment, mapping to domain objects | May be passed to other DTOs or service methods |
| Persistence | `NotificationRecord` | DAO (`RawTableDataAccess` etc.) extracts fields, builds INSERT statements | Rows in Hive/Oracle tables |
| Logging / Monitoring | `NotificationRecord` | `toString()` used in log statements | Log entries for audit/debug |

# Integrations
- **Notification Handler** – receives HTTP POST, parses JSON into `NotificationRecord`.
- **DAO Layer** – `RawTableDataAccess` reads fields via getters to construct persistence payloads.
- **Logging Framework** – `toString()` output sent to SLF4J/Log4j.
- **Potential downstream services** – any component that consumes the persisted records (e.g., analytics pipelines).

# Operational Risks
- **Missing/invalid fields** – deserialization may produce `null` values; downstream DAO may generate malformed SQL. *Mitigation*: schema validation before object creation.
- **Large IMSI list** – unbounded `ArrayList` may cause memory pressure. *Mitigation*: enforce size limits in handler.
- **Date parsing assumptions** – `getStrEventDate()` assumes ISO format; malformed timestamps cause `StringIndexOutOfBoundsException`. *Mitigation*: guard with length check or use a date library.
- **Boolean to String conversion** – `setSbsMember(null)` stores empty string, which may be interpreted incorrectly downstream. *Mitigation*: define explicit sentinel value or use nullable field.

# Usage
```java
// Example unit‑test snippet
String json = "{ \"initiatorId\":\"user1\", \"eventDateTime\":\"2024-11-30T12:34:56Z\", ... }";
ObjectMapper mapper = new ObjectMapper();
NotificationRecord rec = mapper.readValue(json, NotificationRecord.class);

// Debug print
System.out.println(rec);
```
Run within the notification handler context or as a standalone JUnit test.

# Configuration
- No environment variables or external config files are referenced directly by this class.
- Relies on the JSON deserializer configuration (e.g., Jackson `ObjectMapper`) defined elsewhere in the application.

# Improvements
1. **Replace manual date substring with robust parsing** – use `java.time.LocalDateTime` to extract date safely.
2. **Implement immutable variant** – builder pattern to create thread‑safe, read‑only instances, reducing accidental mutation in multi‑threaded pipelines.