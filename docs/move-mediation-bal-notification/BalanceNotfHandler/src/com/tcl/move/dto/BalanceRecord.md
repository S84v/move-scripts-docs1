# Summary
`BalanceRecord` is a plain‑old‑Java‑object (POJO) that encapsulates the fields of a balance‑update event received from the Move API. It provides mutable getters/setters, parses the ISO‑8601 timestamp into a `java.sql.Timestamp`, and supplies string representations for logging and downstream persistence.

# Key Components
- **Class `com.tcl.move.dto.BalanceRecord`**
  - Private fields: identifiers, event metadata, subscriber data, `double balance`, and two `SimpleDateFormat` instances.
  - Public accessor methods for each field.
  - `setEventDateTime(String)` – parses ISO‑8601 string (`yyyy-MM-dd'T'HH:mm:ss.SSS'Z'`) into `Timestamp`.
  - `getStrEventDateTime()` / `getStrEventDate()` – formatted string views of `eventDateTime`.
  - Overridden `toString()` – concatenates all fields for diagnostic output.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| API consumer | JSON payload (balance‑update event) | JSON mapper populates a `BalanceRecord` instance via setters; `setEventDateTime` converts string → `Timestamp`. | In‑memory DTO passed to `RawTableDataAccess.insertInBalanceRawTable` for Hive insertion; optionally to Kafka producer. |
| Logging | DTO | `toString()` invoked by callers for audit logs. | Human‑readable log line. |
| Persistence | DTO | DAO extracts field values for prepared INSERT statement. | Row in Hive table `mnaas.api_balance_update`. |

# Integrations
- **`RawTableDataAccess`** – consumes `BalanceRecord` to construct INSERT parameters.
- **`Constants`** – may reference error categories (`Constants.ERROR_RAW`) when DAO catches exceptions.
- **Kafka producer** (outside this file) may serialize the DTO for the `Constants.BALANCE` topic.
- **JSON parser** (e.g., Jackson) creates the DTO from the inbound API message.

# Operational Risks
- **Thread‑unsafe `SimpleDateFormat`** – shared mutable instances can corrupt parsing/formatting under concurrent use.  
  *Mitigation*: synchronize access or replace with `java.time.format.DateTimeFormatter`.
- **ParseException handling** – on malformed timestamp the DTO silently sets `eventDateTime` to `null`, potentially causing downstream `NULL` inserts.  
  *Mitigation*: surface parsing errors via a checked exception or validation layer.
- **Mutable DTO** – callers can alter state after persistence, leading to inconsistent audit trails.  
  *Mitigation*: make DTO immutable or enforce defensive copying.
- **Precision loss** – `double` for monetary balance may introduce rounding errors.  
  *Mitigation*: use `java.math.BigDecimal` for financial values.

# Usage
```java
// Example unit‑test or debugging snippet
BalanceRecord rec = new BalanceRecord();
rec.setInitiatorId("SYS");
rec.setTransactionId("TX12345");
rec.setSourceId("API");
rec.setTenancyId("T001");
rec.setEventId("E1001");
rec.setParentEventId("PE1000");
rec.setEntityType("ACCOUNT");
rec.setBusinessUnitId("BU01");
rec.setEventClass("BALANCE_UPDATE");
rec.setEventDateTime("2023-08-15T10:30:45.123Z"); // parses to Timestamp
rec.setBusinessUnit("Consumer");
rec.setEventName("BalanceRefresh");
rec.setMsisdn("919876543210");
rec.setSim("8901234567890123456");
rec.setImsi("310150123456789");
rec.setBalance(125.75);

System.out.println(rec); // invokes toString()
```
Run/debug by stepping through the setter calls; verify `rec.getEventDateTime()` yields a non‑null `Timestamp`.

# configuration
- No environment variables are read directly by this class.  
- Relies on two hard‑coded date patterns:
  - `yyyy-MM-dd'T'HH:mm:ss.SSS'Z'` (ISO‑8601 with milliseconds, UTC)
  - `yyyy-MM-dd` (date‑only)

# Improvements
1. **Thread safety & modern API** – Replace `SimpleDateFormat` with `DateTimeFormatter` and store `java.time.Instant` or `OffsetDateTime` instead of `Timestamp`.  
2. **Immutability & validation** – Convert to a final class with a builder pattern; validate required fields and timestamp format at construction, throwing a domain‑specific exception on failure.