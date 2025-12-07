# Summary
`OCSRecord` is a plain‑old‑Java‑Object (POJO) that models an OCS (Online Charging System) event received as JSON by the Move Mediation Notification Handler (MNAAS). It captures all raw fields extracted from the inbound payload, provides conventional getters/setters, a helper to obtain the date portion of `eventDateTime`, and a safe conversion for the `duration` field. Instances are handed off to DAO components for persistence into Hive/Oracle tables and may be logged or inspected during troubleshooting.

# Key Components
- **Class `OCSRecord`**
  - Private member variables for core OCS attributes (e.g., `initiatorId`, `eventId`, `msisdn`, `duration`, etc.).
  - Standard getter and setter methods for each field.
  - `getStrEventDate()` – returns the first 10 characters of `eventDateTime` (YYYY‑MM‑DD) or empty string if null.
  - `setDuration(String)` – parses a string to `double`; defaults to `0.0` on error.
  - Overridden `toString()` – concatenates all fields into a readable representation.

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | JSON payload from external API (OCS event). Parsed by a JSON mapper (e.g., Jackson) into an `OCSRecord` instance. |
| **Processing** | Business logic may read/write fields, compute derived values, or enrich the record. |
| **Output** | The populated `OCSRecord` is passed to DAO classes (e.g., `RawTableDataAccess`) which translate the POJO into Hive/Oracle insert statements. |
| **Side Effects** | None within the POJO itself; side effects occur in downstream DAO or logging components. |
| **External Services** | Not directly invoked; relies on external JSON source and downstream persistence services. |

# Integrations
- **Notification Handler** – `OCSRecord` is instantiated in the notification handling pipeline when an OCS‑type message is detected.
- **DAO Layer** – `RawTableDataAccess` or similar DAO classes accept `OCSRecord` objects to generate SQL/Hive DML.
- **Logging/Monitoring** – `toString()` is used for debug logs and audit trails.
- **Other DTOs** – May be transformed into generic `NotificationRecord` objects for unified processing.

# Operational Risks
- **Parsing Errors** – `setDuration(String)` silently swallows conversion failures, potentially hiding data quality issues. *Mitigation*: log the exception and flag the record as invalid.
- **Date Substring Assumption** – `getStrEventDate()` assumes `eventDateTime` is at least 10 characters; malformed timestamps cause `StringIndexOutOfBoundsException`. *Mitigation*: validate length before substring or use a date‑time library.
- **Mutable State** – All fields are mutable; concurrent modifications could lead to inconsistent records if shared across threads. *Mitigation*: treat instances as thread‑local or make the class immutable.
- **Missing Validation** – No null‑checks for required fields before persistence, risking DB constraint violations. *Mitigation*: introduce a validation step prior to DAO calls.

# Usage
```java
// Example: creating and populating an OCSRecord
OCSRecord rec = new OCSRecord();
rec.setInitiatorId("SYS01");
rec.setTransactionId("TX12345");
rec.setEventDateTime("2024-11-30T14:22:10Z");
rec.setDuration("125.6");   // parsed to double
rec.setMsisdn("15551234567");

// Debug output
System.out.println(rec);               // invokes toString()
System.out.println(rec.getStrEventDate()); // "2024-11-30"

// Pass to DAO
rawTableDao.insertOCSRecord(rec);
```
*Debugging*: Set a breakpoint in any getter/setter or in `setDuration` to verify conversion logic.

# Configuration
The POJO itself does not read configuration files or environment variables. It relies on external components for:
- JSON mapping configuration (e.g., Jackson `ObjectMapper` settings).
- Database connection parameters (managed by DAO layer configuration files or environment variables).

# Improvements
1. **Replace raw `String` date handling with `java.time` types** – store `eventDateTime` as `OffsetDateTime` and provide a formatted date method, eliminating fragile substring logic.
2. **Introduce validation and immutability** – use a builder pattern (or Lombok `@Builder`/`@Value`) to create fully validated, immutable instances, and surface parsing errors instead of silently defaulting.