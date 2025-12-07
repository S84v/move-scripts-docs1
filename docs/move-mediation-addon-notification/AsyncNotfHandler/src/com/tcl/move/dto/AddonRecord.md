# Summary
`AddonRecord` is a plain‑old‑Java‑Object (POJO) that models the payload of an asynchronous move‑notification JSON message. It encapsulates all event, resource, billing, and supplemental fields required by downstream components (DAO, service, logging) to persist a move event into the Hive raw table.

# Key Components
- **class `AddonRecord`**
  - Private fields for 38 attributes (identifiers, timestamps, resource details, billing data, auxiliary codes).
  - Public getter/setter pairs for each attribute.
  - `getEventDate()` – derived accessor that returns the date portion of `eventDateTime`.
  - Overridden `toString()` – concatenates all fields into a single log‑friendly string.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. JSON ingestion | Raw JSON from API/Kafka (`TOPIC_ASYNC`) | Deserialization (e.g., Jackson) → populates `AddonRecord` via setters | In‑memory `AddonRecord` instance |
| 2. Business logic | `AddonRecord` instance | Validation / enrichment (outside this class) | Possibly modified `AddonRecord` |
| 3. Persistence | `AddonRecord` | `RawTableDataAccess` extracts fields, builds a prepared‑statement batch using the INSERT SQL from `MOVEDAO.properties` | Rows inserted into Hive table `mnaas.api_notification_addon` |
| 4. Logging / Auditing | `AddonRecord` | `toString()` used by loggers | Human‑readable audit entry |

# Integrations
- **AsyncNotfHandler** – receives Kafka messages, creates `AddonRecord` objects.
- **RawTableDataAccess** – consumes `AddonRecord` to write to Hive/Impala.
- **Constants** – provides identifiers (`ASYNC`, `TOPIC_ASYNC`) that drive message routing.
- **JDBCConnection** – supplies the Hive connection used by the DAO that persists `AddonRecord`.
- **External services** – JSON producer (API gateway), Kafka broker, Hive/Impala cluster.

# Operational Risks
- **Null / malformed fields** – getters assume non‑null strings (e.g., `getEventDate()` may throw `StringIndexOutOfBoundsException`).  
  *Mitigation*: Validate input before object creation; add defensive checks.
- **Date handling** – `eventDateTime` is a raw string; substring logic is locale‑agnostic and brittle.  
  *Mitigation*: Store timestamps as `java.time.Instant` or `OffsetDateTime` and format as needed.
- **Mutable state** – all fields are mutable, enabling accidental modification after persistence.  
  *Mitigation*: Consider making the class immutable (builder pattern) or expose only getters after construction.
- **Missing fields in DB insert** – DAO relies on exact field order; any future field addition must be synchronized with SQL.  
  *Mitigation*: Use named parameters or ORM mapping.

# Usage
```java
// Example: constructing from a JSON map (pseudo‑code)
AddonRecord rec = new AddonRecord();
rec.setInitiatorId(json.get("initiatorId"));
rec.setTransactionId(json.get("transactionId"));
...
// Pass to DAO
RawTableDataAccess dao = new RawTableDataAccess();
dao.insertBatch(Collections.singletonList(rec));
```
*Debug*: Set a breakpoint in any setter or in `toString()`; inspect the object before DAO invocation.

# configuration
- No environment variables are read directly by this class.  
- Relies on external configuration:
  - `Constants.java` for topic/identifier constants.  
  - `MOVEDAO.properties` for the INSERT SQL used by `RawTableDataAccess`.  

# Improvements
1. **Add validation logic** – implement a `validate()` method that checks mandatory fields, date format, and numeric ranges; invoke before DAO persistence.  
2. **Modernize date handling** – replace `String eventDateTime` with `OffsetDateTime eventDateTime` and provide ISO‑8601 parsing/formatting; update `getEventDate()` accordingly.