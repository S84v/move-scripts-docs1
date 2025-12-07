# Summary
`NotificationConsumer` is a singleton Kafka consumer that continuously polls a set of MOVE‑mediation topics, dispatches each record to the appropriate service (`SwapService` or `NotificationService`) based on the record key, and commits offsets manually. It is the runtime entry point for processing inbound notification events in the production MOVE mediation pipeline.

# Key Components
- **class `NotificationConsumer`**
  - `getInstance()` – creates the singleton and initializes the Kafka consumer.
  - `createConsumer()` – builds a `KafkaConsumer<String,String>` with explicit configuration properties and subscribes to all relevant MOVE topics.
  - `runConsumer()` – infinite poll loop; logs, routes records, and performs synchronous offset commits.
  - `shutdown()` – triggers a `WakeupException` to break the poll loop and close the consumer.
- **static fields**
  - `logger` – Log4j logger.
  - `singleton` – holds the sole instance.
  - `consumer` – shared `KafkaConsumer`.
- **instance fields**
  - `swapService` – `SwapService` instance for SIM/MSISDN swap handling.
  - `notificationService` – `NotificationService` instance for all other notification types.

# Data Flow
| Stage | Source / Destination | Description |
|-------|----------------------|-------------|
| Input | Kafka broker (servers from `kafka_server`/`kafka_port` system properties) | Consumes messages from topics listed in `Constants` (e.g., `TOPIC_SIM_SWAP`, `TOPIC_MSISDN_SWAP`, etc.). |
| Processing | In‑process services (`SwapService`, `NotificationService`) | Dispatches based on `record.key()` to the matching method (e.g., `performSIMSwap`, `processOtaRequest`). |
| Output | Downstream systems invoked by the services (e.g., external APIs, databases) | Not shown in this file; side effects are performed inside the service methods. |
| Commit | Kafka broker | Synchronous `consumer.commitSync()` after each poll batch. |
| Logging | Log4j appender | Detailed per‑record logs and error handling. |

# Integrations
- **Kafka** – consumes from multiple MOVE topics; uses `org.apache.kafka.clients.consumer.KafkaConsumer`.
- **Constants** – topic names and key identifiers are defined in `com.tcl.move.constants.Constants`.
- **SwapService** – handles SIM and MSISDN swap logic.
- **NotificationService** – processes status updates, OTA, location updates, subscriber creation, OCS, eSIM, SIM‑IMEI, and bulk eSIM requests.
- **Log4j** – central logging framework for operational visibility.

# Operational Risks
- **Infinite loop without graceful shutdown** – process must be terminated externally; mitigated by invoking `shutdown()` via a signal handler or external orchestrator.
- **Manual offset commit on every poll** – if processing of a single record fails, the entire batch may be re‑processed, leading to duplicate handling; mitigate by adding per‑record error handling and selective commit.
- **Hard‑coded consumer group ID** – all instances share `"Notification Consumer"`; scaling may cause rebalancing overhead; consider externalizing the group ID.
- **Static system‑property configuration** – missing `kafka_server`/`kafka_port` causes `NullPointerException`; validate properties at startup.
- **No back‑pressure or throttling** – high message volume could overwhelm downstream services; implement rate limiting or async processing.

# Usage
```bash
# Set required system properties
export KAFKA_SERVER=broker1.example.com
export KAFKA_PORT=9092

# Run from the main application (e.g., NotificationHandler) or directly:
java -Dkafka_server=$KAFKA_SERVER -Dkafka_port=$KAFKA_PORT \
     -cp <classpath> com.tcl.move.service.NotificationConsumer
```
For debugging, attach a debugger to the `runConsumer` method or replace the infinite loop with a bounded iteration count.

# Configuration
- **System properties**
  - `kafka_server` – Kafka bootstrap host.
  - `kafka_port` – Kafka bootstrap port.
- **Consumer properties (hard‑coded)**
  - Deserializers: `StringDeserializer`.
  - `fetch.min.bytes` = 512.
  - `enable.auto.commit` = false.
  - `auto.commit.interval.ms` = 1000.
  - `session.timeout.ms` = 60000.
  - `max.poll.interval.ms` = 900000.
  - `max.poll.records` = 50.
  - `group.id` = `"Notification Consumer"`.

# Improvements
1. **Externalize consumer configuration** – load properties from a YAML/JSON file or environment variables to allow runtime tuning without code changes.
2. **Add robust error handling** – wrap each service call in try/catch, record failed offsets, and commit only successfully processed records to avoid duplicate processing.