# Summary
`NotificationService` processes a Balance Update request received as a JSON string. It parses the JSON into a `BalanceRecord`, persists the record to the raw balance table via `RawTableDataAccess`, and handles failures by sending alert e‑mails through `MailService`. It is invoked by the batch entry point (`BalanceNotificationHandler`) and the Kafka consumer (`NotificationConsumer`).

# Key Components
- **class `NotificationService`**
  - `private static final Logger logger` – Log4j logger.
  - `RawTableDataAccess rawTableDAO` – DAO for raw‑table persistence.
  - `MailService mailService` – Service for sending alert e‑mails.
  - `public void processBalanceRequest(String balanceJSON)` – Core method that:
    1. Parses JSON → `BalanceRecord`.
    2. Inserts record into raw table.
    3. Catches `DatabaseException` (raw‑table errors) → sends alert mail.
    4. Catches `ParseException` (JSON parsing errors) → sends alert mail.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Invocation | JSON string (`balanceJSON`) from file or Kafka | `JSONParseUtils.parseBalanceJSON` → `BalanceRecord` | `BalanceRecord` instance |
| Persistence | `BalanceRecord` | `rawTableDAO.insertInBalanceRawTable` | Row inserted into **RAW_BALANCE** table (or DB exception) |
| Error handling – DB | `DatabaseException` with `errorType == Constants.ERROR_RAW` | Build alert body, call `mailService.sendAlertMail` | Alert e‑mail sent; log entry |
| Error handling – Parse | `ParseException` | Build alert body, call `mailService.sendAlertMail` | Alert e‑mail sent; log entry |
| Logging | All paths | `logger.error` / `logger.info` | Log entries in application logs |

External services:
- **Database** – accessed via `RawTableDataAccess`.
- **SMTP server** – used by `MailService` to send alert e‑mails.

# Integrations
- **BalanceNotificationHandler** – Calls `processBalanceRequest` for each line in a re‑process file or for each Kafka record.
- **NotificationConsumer** – Retrieves messages from the **Balance** Kafka topic and forwards the payload to `NotificationService`.
- **JSONParseUtils** – Utility for converting JSON to `BalanceRecord`.
- **RawTableDataAccess** – DAO layer that encapsulates JDBC/ORM interaction with the raw balance table.
- **MailService** – Sends alert e‑mails; relies on system properties for SMTP configuration.

# Operational Risks
- **Uncaught Runtime Exceptions** – Only `DatabaseException`, `ParseException`, and `MailException` are handled; other runtime exceptions will propagate and may terminate the consumer thread. *Mitigation*: Add a generic `catch (Exception e)` block with logging and optional alert.
- **Mail Flooding** – Repeated failures can generate a high volume of alert e‑mails. *Mitigation*: Implement rate‑limiting or aggregate alerts.
- **Blocking I/O** – `mailService.sendAlertMail` is synchronous; a slow SMTP server can delay processing. *Mitigation*: Offload mail sending to an async executor or queue.
- **Hard‑coded DAO instance** – No dependency injection; testing and swapping implementations is difficult. *Mitigation*: Refactor to use constructor injection.

# Usage
```java
// Unit test / debugging example
public static void main(String[] args) {
    NotificationService svc = new NotificationService();
    String sampleJson = "{\"accountId\":\"12345\",\"balance\":100.0,\"timestamp\":\"2025-12-01T12:00:00Z\"}";
    svc.processBalanceRequest(sampleJson);
}
```
Run the full batch:
```bash
java -cp <classpath> com.tcl.move.main.BalanceNotificationHandler \
    -mode file -input /data/reprocess.txt
```
or start the Kafka consumer:
```bash
java -cp <classpath> com.tcl.move.main.BalanceNotificationHandler -mode kafka
```

# Configuration
- **System properties / environment variables** used by `MailService` (SMTP host, port, user, password). Typically set in `application.properties` or passed as `-Dmail.smtp.host=...`.
- **Database connection** details are read by `RawTableDataAccess` (JDBC URL, credentials) from the same property source.
- **Constants.ERROR_RAW** defined in `com.tcl.move.constants.Constants`.

# Improvements
1. **Introduce Dependency Injection** – Pass `RawTableDataAccess` and `MailService` via constructor to enable mocking and configuration flexibility.
2. **Add Generic Exception Handling & Alert Throttling** – Catch all unexpected exceptions, log stack trace, and implement a back‑off or aggregation mechanism for alert e‑mails to prevent alert storms.