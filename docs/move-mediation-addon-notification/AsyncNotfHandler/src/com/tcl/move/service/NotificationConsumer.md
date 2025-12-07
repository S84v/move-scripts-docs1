# Summary
`NotificationConsumer` is a singleton Kafka consumer that continuously polls the `TOPIC_ASYNC` queue for move‑mediation add‑on notification messages. It filters records by key, parses the JSON payload into `AddonRecord` objects, forwards successful records to `NotificationService.processAsyncRequest`, and sends alert e‑mails via `MailService` on parsing failures or daily health‑check pings.

# Key Components
- **class `NotificationConsumer`**
  - Private static logger.
  - Static singleton instance (`singleton`) and static Kafka `Consumer<String,String>` (`consumer`).
  - Private constructor to enforce singleton usage.
  - `public static NotificationConsumer getInstance()` – creates singleton and initializes Kafka consumer.
  - `private static Consumer<String,String> createConsumer()` – builds Kafka consumer properties and subscribes to `Constants.TOPIC_ASYNC`.
  - `public void runConsumer()` – main loop: poll, health‑check mail, parse records, collect `AddonRecord`s, invoke `NotificationService`, commit offsets, handle exceptions.
  - `public void shutdown()` – triggers consumer wake‑up for graceful termination.
- **Dependencies**
  - `NotificationService` – downstream processing of parsed `AddonRecord`s.
  - `MailService` – sends alert e‑mails (`sendAlertMail`).
  - `JSONParseUtils.parseAsyncJSON(String)` – converts JSON string to `AddonRecord`.
  - Custom exceptions: `ParseException`, `MailException`.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| Kafka poll | Records from `Constants.TOPIC_ASYNC` (key/value strings) | Filter by key `Constants.ASYNC`; compare offset with `prevOffset` | `AddonRecord` list, error logs |
| JSON parse | `record.value()` (JSON) | `JSONParseUtils.parseAsyncJSON` → may throw `ParseException` | Successful `AddonRecord` added to `recordsToInsert`; on failure, alert e‑mail sent |
| Service call | `recordsToInsert` | `notificationService.processAsyncRequest(recordsToInsert)` | Downstream business handling (DB writes, further messaging) |
| Commit | Current `offset` | `consumer.commitSync()` | Kafka offset persisted |
| Health‑check mail | Daily timer (date change) | Build body, `mailService.sendAlertMail` | Alert e‑mail confirming process liveness |
| Error handling | `WakeupException`, generic `Exception` | Log, close consumer, exit or continue | Process termination or continued operation |

# Integrations
- **Kafka** – consumes from topic defined in `Constants.TOPIC_ASYNC`; uses broker list from system property `kafka_servers_list`.
- **NotificationService** – internal service that likely persists or forwards `AddonRecord`s to downstream systems (e.g., databases, other queues).
- **MailService** – external SMTP server configured via system properties; used for alert and health‑check e‑mails.
- **JSONParseUtils** – utility for JSON deserialization; throws `ParseException` on malformed payloads.
- **Constants** – provides static values such as `TOPIC_ASYNC` and key identifier `ASYNC`.

# Operational Risks
- **Offset management** – manual offset tracking (`prevOffset`) may cause duplicate or missed records if consumer restarts; mitigate by persisting offset externally or relying on Kafka’s committed offsets.
- **Infinite loop without back‑off** – tight `while(true)` may hammer Kafka on empty polls; consider adding sleep/back‑off on consecutive empty polls.
- **Single‑threaded processing** – large volume could cause latency; scale by partitioning topic and running multiple consumer instances with distinct group IDs.
- **Mail flood** – on continuous parse failures, rapid e‑mail generation could overwhelm SMTP; implement rate‑limiting or aggregate alerts.
- **Hard‑coded consumer group** – `"Notification Consumer"` may clash with other services; make configurable.

# Usage
```bash
# Set required system properties
export kafka_servers_list="broker1:9092,broker2:9092"
# (Optional) configure SMTP via JavaMail system properties as required by MailService

# Run the consumer (typically invoked from AsyncNotificationHandler.main)
java -cp <classpath> com.tcl.move.service.NotificationConsumer
```
For debugging, attach a debugger to `runConsumer()` or invoke `getInstance().runConsumer()` from a test harness.

# configuration
- **System property** `kafka_servers_list` – comma‑separated list of Kafka bootstrap servers.
- **MailService** relies on JavaMail system properties (e.g., `mail.smtp.host`, `mail.smtp.port`, `mail.smtp.auth`, etc.) defined elsewhere in the application.
- **Constants** (compiled class) provides `TOPIC_ASYNC` and `ASYNC` key values.

# Improvements
1. Replace manual `prevOffset` tracking with Kafka’s committed offset mechanism or external durable store to guarantee exactly‑once processing across restarts.
2. Introduce configurable health‑check interval and e‑mail rate‑limiting; externalize consumer group ID and poll parameters to a properties file for easier tuning.