# Summary
`NotificationService` implements the business‑logic layer for processing various MOVE‑mediation notification types (status updates, OTA, location updates, subscriber creation, SIM‑IMEI mapping, OCS bypass, E‑SIM, bulk E‑SIM). Each public method parses an incoming JSON payload, persists a raw record, optionally updates inventory tables, and handles database or parsing failures by inserting into a fail table and/or sending alert e‑mails via `MailService`.

# Key Components
- **Class `NotificationService`**
  - `processStatusUpdateRequest(String)` – parses status‑update JSON, writes raw record, updates SIM status, adds IMSIs, error handling with fail table & mail alerts.
  - `processOtaRequest(String)` – parses OTA JSON, writes raw record, error handling similar to status update.
  - `processLocationUpdateRequest(String, String)` – handles “UPDATE” or “RAW” location messages, writes appropriate raw table, mail alerts on DB/parse errors.
  - `processSubsCreationRequest(String)` – parses subscriber‑creation JSON, writes raw record, fail‑table fallback, mail alerts.
  - `processSIMIMEIRequest(String)` – parses SIM‑IMEI JSON, writes raw record, updates IMEI in inventory, mail alerts on DB/parse errors.
  - `processOCSRequest(String)` – parses OCS bypass JSON, writes raw record, mail alerts on DB/parse errors.
  - `processESimRequest(String)` – parses single E‑SIM JSON, writes raw record, mail alerts on DB/parse errors.
  - `processBulkESimRequest(String)` – identical to `processESimRequest` for bulk payloads.
- **Dependencies**
  - `RawTableDataAccess` – DAO for raw and fail table inserts.
  - `SIMInventoryDataAccess` – DAO for status, IMSI, and IMEI updates.
  - `MailService` – sends alert e‑mails on unrecoverable errors.
  - `JSONParseUtils` – static parsers for each notification type.
  - `Constants.ERROR_RAW` – error type identifier for raw‑table failures.
  - `Logger` – log4j logging.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | JSON payload (string) from Kafka consumer or batch file | `JSONParseUtils` → DTO (`NotificationRecord`, `LocationUpdateRecord`, etc.) | DTO instance |
| 2 | DTO | `RawTableDataAccess.insertIn*` | Row inserted into raw table (DB) |
| 3a (status/IMEI) | DTO | `SIMInventoryDataAccess` updates status/IMEI/IMSIs | Inventory tables updated |
| 4a (failure on raw insert) | `DatabaseException` with `ERROR_RAW` | Insert same DTO into fail table (`insertInRawFail`) | Fail table row |
| 4b (failure on fail insert) | `DatabaseException` | Build alert body & `MailService.sendAlertMail` | Alert e‑mail sent |
| 5 (parse error) | `ParseException` | Build alert body & `MailService.sendAlertMail` | Alert e‑mail sent |
| Logging | Throughout | `logger.error/info` | Log entries in application logs |

External services:
- Relational DB (raw, fail, inventory tables)
- SMTP server (via `MailService`)

# Integrations
- **`NotificationConsumer`** (Kafka) → invokes the appropriate `process*` method based on record key.
- **`NotificationHandler`** (batch/cron) → reads CSV/flat files, calls the same service methods.
- **DAO layer** (`RawTableDataAccess`, `SIMInventoryDataAccess`) → JDBC/ORM implementations interacting with the MOVE mediation database schema.
- **`MailService`** → JavaMail API, configured via system properties (SMTP host, from/to addresses).

# Operational Risks
- **Database insert failures** – raw table down leads to fail‑table cascade; mitigated by alert e‑mail and manual re‑processing.
- **Fail‑table insert also failing** – loss of audit trail; mitigated by immediate alert and retry mechanisms (not present – should be added).
- **Parsing errors** – malformed JSON results in dropped messages; mitigated by alert and possible dead‑letter queue (not implemented).
- **Hard‑coded fail code (103)** – limited error taxonomy; consider externalizing error codes.
- **Synchronous mail send** – could block processing thread; consider async mail queue.

# Usage
```java
// Unit test / debugging example
NotificationService svc = new NotificationService();
String json = "{\"sim\":\"SIM123\",\"msisdn\":\"1234567890\",...}";
svc.processStatusUpdateRequest(json);
```
- Run via the Kafka consumer (`NotificationConsumer`) or batch driver (`NotificationHandler.main`) which supplies the JSON strings.

# Configuration
- **System properties (MailService)**  
  - `mail.smtp.host` – SMTP server hostname  
  - `mail.from` – sender address  
  - `mail.to` – recipient address(es)  
- **Log4j configuration** – `log4j.properties` for `com.tcl.move.service.NotificationService`.
- **Database connection** – defined in DAO configuration files (JDBC URL, credentials) – not referenced directly in this class.

# Improvements
1. **Introduce retry and circuit‑breaker logic** for raw/fail table inserts and mail sending to reduce manual intervention.
2. **Refactor duplicated error‑handling blocks** into a private helper method (e.g., `handleDatabaseException(record, json, daoInsertMethod)`) to improve maintainability and reduce code churn.