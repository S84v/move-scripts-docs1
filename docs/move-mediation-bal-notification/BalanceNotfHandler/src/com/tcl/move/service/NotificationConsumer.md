# Summary
`NotificationConsumer` is a singleton Kafka consumer that continuously polls the **Balance** topic, filters records with key `BALANCE`, and forwards each message payload to `NotificationService.processBalanceRequest`. It manually manages offsets, commits after processing, and provides a shutdown hook via `consumer.wakeup()`.

# Key Components
- **class `NotificationConsumer`**
  - `private NotificationConsumer()` – private constructor to enforce singleton usage.
  - `public static NotificationConsumer getInstance()` – creates (or recreates) the singleton and initializes the Kafka consumer.
  - `private static Consumer<String,String> createConsumer()` – builds a `KafkaConsumer` with required properties and subscribes to `Constants.TOPIC_BALANCE`.
  - `public void runConsumer()` – infinite poll loop; processes records, logs progress, commits offsets, and handles `WakeupException` and generic exceptions.
  - `public void shutdown()` – triggers a graceful shutdown by invoking `consumer.wakeup()`.
- **static fields**
  - `logger` – Log4j logger.
  - `singleton` – holds the sole instance.
  - `consumer` – shared Kafka consumer instance.
- **instance field**
  - `notificationService` – `NotificationService` used to process balance notifications.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Consumer creation | System property `kafka_servers_list` | Build `Properties`, instantiate `KafkaConsumer`, subscribe to `Constants.TOPIC_BALANCE` | Ready consumer object |
| 2. Poll loop | Kafka broker messages (key/value strings) | `consumer.poll(50)` → iterate `ConsumerRecord`s → filter `record.key()==Constants.BALANCE` → deduplicate by offset → call `notificationService.processBalanceRequest(record.value())` | Business processing of balance payload |
| 3. Commit | Processed offset | `consumer.commitSync()` if any offset processed | Offset persisted in Kafka |
| 4. Shutdown | External trigger (`shutdown()` call) | `consumer.wakeup()` → `WakeupException` caught → `consumer.close()` → `System.exit(1)` | Consumer terminated, JVM exit |

External services:
- **Kafka cluster** (bootstrap servers from `kafka_servers_list`).
- **NotificationService** (internal service, may call downstream APIs, DB, etc.).

# Integrations
- **`BalanceNotificationHandler`** – entry point that likely invokes `NotificationConsumer.getInstance().runConsumer()`.
- **`Constants`** – provides `TOPIC_BALANCE` and key constant `BALANCE`.
- **`NotificationService`** – business layer handling the actual balance request logic.
- **Kafka** – source of messages; consumer group ID `"Notification Consumer"`.

# Operational Risks
- **Infinite loop without back‑off** – continuous tight polling may cause high CPU usage if no messages; mitigate by increasing poll timeout or adding sleep on empty batches.
- **Offset management** – manual commit after each batch; if processing fails after commit, messages are lost. Mitigate by committing only after successful processing of each record or using transactional processing.
- **Singleton recreation** – `getInstance()` always creates a new instance, discarding any existing consumer; could lead to resource leaks. Mitigate by checking `if (singleton == null)` before recreation.
- **Hard exit on WakeupException** – `System.exit(1)` terminates the JVM, preventing graceful shutdown of other components. Replace with proper shutdown sequence.
- **Hard‑coded consumer configs** – values (e.g., fetch size, poll interval) are static; changes require code redeploy. Externalize to configuration.

# Usage
```java
public static void main(String[] args) {
    NotificationConsumer consumer = NotificationConsumer.getInstance();
    Runtime.getRuntime().addShutdownHook(new Thread(consumer::shutdown));
    consumer.runConsumer();   // blocks indefinitely
}
```
*Debug*: set `kafka_servers_list` JVM property, attach a debugger to `runConsumer()` loop, and inspect `record.value()`.

# Configuration
- **System property** `kafka_servers_list` – comma‑separated list of Kafka bootstrap servers.
- **Constants** (via `com.tcl.move.constants.Constants`):
  - `TOPIC_BALANCE` – Kafka topic name.
  - `BALANCE` – expected record key.
- No external config files referenced directly; all other consumer settings are hard‑coded.

# Improvements
1. **Singleton Guard** – modify `getInstance()` to instantiate only once:
   ```java
   if (singleton == null) {
       singleton = new NotificationConsumer();
       consumer = createConsumer();
   }
   return singleton;
   ```
2. **Graceful Shutdown & Exit Strategy** – replace `System.exit(1)` with a controlled stop flag, allow the main thread to exit cleanly, and ensure other services are closed. Also add configurable poll timeout and back‑off when `records.isEmpty()`.