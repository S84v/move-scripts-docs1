# Summary
`SwapService` processes SIM‑swap and MSISDN‑swap notification JSON payloads. It parses the payload, persists a raw record, retrieves related active inventory, and writes aggregation (swap) records. Errors are handled by inserting into a fail table and/or sending alert e‑mails via `MailService`.

# Key Components
- **class `SwapService`**
  - `performSIMSwap(String simSwapJSON)`: handles SIM swap workflow.
  - `performMSISDNSwap(String msisdnSwapJSON)`: handles MSISDN swap workflow.
- **Dependencies**
  - `RawTableDataAccess` – DAO for raw and raw‑fail tables.
  - `AggregationDataAccess` – DAO for aggregation (swap) table and lookup queries.
  - `MailService` – utility to send alert e‑mails.
  - `JSONParseUtils` – static parsers for SIM and MSISDN swap JSON.
  - `Constants` – error‑type string literals.
  - `NotificationRecord`, `SwapRecord` – DTOs representing raw and aggregation rows.
  - `ParseException`, `DatabaseException`, `MailException` – custom exception types.

# Data Flow
1. **Input**: JSON string (`simSwapJSON` or `msisdnSwapJSON`) received from upstream notification pipeline (Kafka consumer).  
2. **Parse**: `JSONParseUtils` → `NotificationRecord`.  
3. **Persist Raw**: `rawTableDAO.insertInRawTable(record)`.  
4. **Lookup**: `aggrTableDAO.getSIMSwapData` / `getMSISDNSwapData` using `record.getBeforeValue()`.  
5. **Decision**  
   - If lookup empty → set fail code/reason, insert into raw‑fail table.  
   - Else → populate `SwapRecord` list, set swap attributes, `aggrTableDAO.insertInSwapTable`.  
6. **Error Handling**  
   - `ParseException` → send alert mail.  
   - `DatabaseException` → based on `errorType` (RAW, FAIL, SWAP) insert into fail table or send alert mail.  
7. **Side Effects**: DB writes (raw, raw‑fail, swap tables) and outbound SMTP e‑mail.

# Integrations
- **Upstream**: `NotificationConsumer` (Kafka) forwards JSON payloads to `SwapService`.  
- **Database**: `RawTableDataAccess` and `AggregationDataAccess` interact with MOVE‑mediation schema (raw, raw_fail, swap tables).  
- **Alerting**: `MailService` uses system properties (`mail.smtp.host`, `mail.from`, `mail.to`) to send SMTP alerts.  
- **Utility**: `JSONParseUtils` shared across notification services for payload deserialization.

# Operational Risks
- **Parsing failures** → alerts generated; risk of alert flooding if payload schema changes. *Mitigation*: versioned schema validation, rate‑limit alerts.  
- **Database insert failures** → fallback to fail table; if fail table also fails, only mail is sent, risking data loss. *Mitigation*: monitor DB health, implement retry/back‑off, ensure fail table high availability.  
- **SMTP misconfiguration** → alert mails not sent, silent failures. *Mitigation*: health‑check SMTP endpoint, alert on `MailException`.  
- **Hard‑coded DAO instances** → no dependency injection; testing and resource management limited. *Mitigation*: refactor to use DI container.

# Usage
```java
SwapService service = new SwapService();

// SIM swap example
String simJson = "{\"eventDateTime\":\"2025-11-30T12:00:00Z\", ...}";
service.performSIMSwap(simJson);

// MSISDN swap example
String msisdnJson = "{\"eventDateTime\":\"2025-11-30T12:05:00Z\", ...}";
service.performMSISDNSwap(msisdnJson);
```
For debugging, set Log4j level to DEBUG for `com.tcl.move.service.SwapService`.

# Configuration
- **System properties (MailService)**  
  - `mail.smtp.host` – SMTP server address.  
  - `mail.from` – Sender e‑mail address.  
  - `mail.to` – Recipient list (comma‑separated).  
- **Log4j** – `log4j.properties` controls logger `com.tcl.move.service.SwapService`.  
- **Database** – Connection details are managed inside `RawTableDataAccess` and `AggregationDataAccess` (typically via a properties file or JNDI).

# Improvements
1. **Introduce Dependency Injection** – replace direct DAO and MailService instantiation with injectable interfaces to enable unit testing and resource pooling.  
2. **Consolidate Error Handling** – create a reusable method to build and send alert mails, reducing duplicated code and ensuring consistent alert content.