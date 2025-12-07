# Summary
`NotificationService` receives a list of `AddonRecord` objects representing asynchronous move‑mediation notifications, attempts to persist them to the raw database table via `RawTableDataAccess`, and on database insertion failures of type `ERROR_RAW` sends an alert e‑mail through `MailService`. It is invoked by the notification consumer or re‑process handler after successful JSON parsing.

# Key Components
- **class `NotificationService`**
  - `processAsyncRequest(List<AddonRecord> records)`: core method that writes records to the raw table and handles error‑mailing.
- **dependencies**
  - `RawTableDataAccess rawTableDAO`: DAO responsible for `insertInAddonRawTable`.
  - `MailService mailService`: utility for sending alert e‑mails.
  - `Logger logger`: Log4j logger for operational messages.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Invocation | `List<AddonRecord> records` (parsed from JSON) | `rawTableDAO.insertInAddonRawTable(records)` | Records persisted in raw DB table |
| Failure (DatabaseException) | `DatabaseException e` with `errorType = Constants.ERROR_RAW` | Build alert body, call `mailService.sendAlertMail(mailBody)` | Alert e‑mail sent to configured recipients; error logged |
| Failure (MailException) | `MailException mailEx` | Log mail‑send failure | No further action |

External services:
- Relational database accessed via `RawTableDataAccess`.
- SMTP server accessed via `MailService`.

# Integrations
- **`NotificationConsumer`**: polls Kafka `TOPIC_ASYNC`, parses payloads, and calls `NotificationService.processAsyncRequest`.
- **`AsyncNotificationHandler`** (re‑process mode): reads static file, builds `List<AddonRecord>`, and forwards to `NotificationService`.
- **`MailService`**: shared utility for all alert e‑mail generation.
- **`RawTableDataAccess`**: DAO used by other services for raw table CRUD operations.

# Operational Risks
- **Database insertion failure not of type `ERROR_RAW`** → no alert is generated; risk of silent data loss. *Mitigation*: broaden error handling or add generic alert.
- **Mail service outage** → alert e‑mail may fail, logged only. *Mitigation*: implement retry/back‑off or queue alerts for later delivery.
- **Unbounded list size** → large `records` list could cause memory pressure or transaction timeout. *Mitigation*: batch inserts or limit list size per call.
- **Hard‑coded DAO and MailService instances** → prevents dependency injection for testing. *Mitigation*: refactor to constructor injection.

# Usage
```java
// Unit test / debugging example
List<AddonRecord> sample = Arrays.asList(new AddonRecord(...));
NotificationService svc = new NotificationService();
svc.processAsyncRequest(sample);
```
Run within the application context where Log4j and DB/JMS configurations are loaded (e.g., via `AsyncNotificationHandler.main`).

# configuration
- **Log4j properties**: `log4j.properties` (defines logger behavior).
- **Database connection**: properties read by `RawTableDataAccess` (JDBC URL, user, password).
- **SMTP settings**: system properties accessed by `MailService` (`mail.smtp.host`, `mail.smtp.port`, `mail.smtp.user`, `mail.smtp.password`).

# Improvements
1. **Introduce constructor injection** for `RawTableDataAccess` and `MailService` to enable mock testing and flexible configuration.
2. **Expand error handling** to generate alerts for any `DatabaseException`, not only `ERROR_RAW`, and consider retry logic for transient DB failures.