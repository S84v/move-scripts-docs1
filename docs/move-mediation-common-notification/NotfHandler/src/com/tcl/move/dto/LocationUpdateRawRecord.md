# Summary
`LocationUpdateRawRecord` is a plain‑old‑Java‑Object (POJO) that models a location‑update event received from external APIs in the Move Mediation Notification Handler (MNAAS). It encapsulates all fields extracted from the inbound JSON payload and provides standard getters/setters plus a helper to obtain the date portion of `eventDateTime`. Instances are passed to DAO classes (e.g., `RawTableDataAccess`) for persistence into Hive/Oracle tables.

# Key Components
- **Class `LocationUpdateRawRecord`**
  - Private String fields: `initiatorId`, `transactionId`, `sourceId`, `tenancyId`, `eventId`, `parentEventId`, `entityType`, `entityId`, `eventClass`, `eventDateTime`, `status`, `mscNumber`, `vlrNumber`, `sgsnNumber`, `sgsnAddress`, `imsi`.
  - Public getters/setters for each field.
  - `getStrEventDate()` – returns the first 10 characters of `eventDateTime` (YYYY‑MM‑DD) or empty string if null.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| API Receiver | JSON payload containing location‑update attributes | JSON → `LocationUpdateRawRecord` via Jackson/Gson mapping (outside this class) | Populated POJO instance |
| Business Logic | POJO instance | Passed to DAO (`RawTableDataAccess.insertInRawLocationUpdate`) | Parameterised INSERT executed against Hive/Oracle |
| Logging | POJO fields | `toString()` (inherited from `Object` unless overridden elsewhere) used for audit logs | Log entries |

No direct external services, DB connections, or queues are handled inside this class.

# Integrations
- **`RawTableDataAccess`** – consumes `LocationUpdateRawRecord` to build INSERT statements.
- **JSON deserialization layer** – maps inbound API JSON to this POJO (e.g., using Jackson `ObjectMapper`).
- **Logging framework** – may log the POJO via its getters or default `toString()`.

# Operational Risks
- **Null `eventDateTime`** – `getStrEventDate()` will return empty string; downstream DB may reject null/empty date values. *Mitigation*: Validate presence before persisting.
- **String length mismatches** – fields may exceed DB column sizes, causing `Data truncation` errors. *Mitigation*: Enforce length checks in validation layer.
- **Missing fields** – API changes could introduce new attributes not represented, leading to data loss. *Mitigation*: Version the DTO and update mapping logic accordingly.

# Usage
```java
// Example unit‑test snippet
LocationUpdateRawRecord rec = new LocationUpdateRawRecord();
rec.setInitiatorId("SYS");
rec.setTransactionId("TX12345");
rec.setEventDateTime("2024-11-30T14:22:10Z");
assertEquals("2024-11-30", rec.getStrEventDate());

// Pass to DAO
rawTableDao.insertInRawLocationUpdate(rec);
```
Debug by inspecting field values after JSON deserialization or by stepping through DAO insertion.

# Configuration
No environment variables or external configuration files are referenced directly by this class. Configuration is required only for components that instantiate or persist the POJO (e.g., DB connection properties in DAO).

# Improvements
- **Add `toString()` override** that formats all fields for concise log output, reducing reliance on default `Object.toString()`.
- **Introduce validation method** (`validate()`) that checks mandatory fields and length constraints before DAO persistence.