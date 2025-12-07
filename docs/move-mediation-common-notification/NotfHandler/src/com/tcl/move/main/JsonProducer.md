# Summary
`JsonProducer` is a utility that creates a Kafka producer, reads a hard‑coded JSON payload, and publishes it to the `move‑apinotif‑asynccallback` topic. It is used in the MOVE mediation notification pipeline to inject test or static notification messages into the Kafka bus for downstream consumers.

# Key Components
- **class `JsonProducer`**
  - `TOPIC` – target Kafka topic (static).
  - `BOOTSTRAP_SERVERS` – Kafka broker address.
  - `jsonFile` – static JSON string representing a notification payload.
  - `createProducer()` – builds and returns a configured `KafkaProducer<String,String>`.
  - `runProducer()` – instantiates the producer, logs the payload, sends a single `ProducerRecord` with key `"Notification"` and the JSON value, flushes and closes the producer.
  - `main(String[] args)` – entry point that invokes `runProducer()`.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | None (static fields) | `createProducer()` reads `BOOTSTRAP_SERVERS` and other producer properties. | Configured `KafkaProducer` instance. |
| 2 | `jsonFile` (static JSON string) | `runProducer()` calls `producer.send()` with a `ProducerRecord` (topic, key, value). | Message published to Kafka topic `move‑apinotif‑asynccallback`. |
| 3 | – | `producer.flush()` ensures delivery; `producer.close()` releases resources. | No further output. |

External services: Kafka broker at `10.133.43.94:9092`. No database interaction.

# Integrations
- **Kafka Cluster** – The producer connects to the broker defined in `BOOTSTRAP_SERVERS` and writes to the topic listed in `TOPIC`. Downstream consumers (e.g., notification handlers, OCS bypass services) read from this topic.
- **Other MOVE components** – The JSON schema matches the contract expected by the asynchronous callback consumer (`move‑apinotif‑asynccallback`). No direct code references, but topic naming conventions indicate alignment with other notification producers (e.g., `move‑apinotif‑simswap`, `move‑apinotif‑statuschange`).

# Operational Risks
- **Hard‑coded payload** – Limits usefulness to testing; production runs may inadvertently send stale data. *Mitigation*: externalize payload via file or CLI argument.
- **Static broker address** – Not environment‑aware; changes require code rebuild. *Mitigation*: read from environment variable or configuration service.
- **No retry/back‑off handling** – `producer.send()` uses default async behavior; failures are only printed. *Mitigation*: implement `Callback` with error handling and retry logic.
- **Lack of schema validation** – Malformed JSON will be sent unchecked. *Mitigation*: validate against JSON schema before sending.

# Usage
```bash
# Compile (assuming Maven/Gradle classpath includes Kafka client libs)
javac -cp "path/to/kafka-clients.jar" com/tcl/move/main/JsonProducer.java

# Run
java -cp ".:path/to/kafka-clients.jar" com.tcl.move.main.JsonProducer
```
For debugging, set a breakpoint in `runProducer()` or replace `System.out.println` with a logger.

# Configuration
- **BOOTSTRAP_SERVERS** – hard‑coded string; modify in source to point to target Kafka cluster.
- **TOPIC** – static topic name; change in source if a different destination is required.
- **Producer properties** – defined in `createProducer()` (acks, retries, batch size, linger, buffer memory, timeouts, serializers, client ID).

# Improvements
1. **Externalize configuration** – Load broker list, topic, and payload from environment variables or a properties file to support multiple environments without recompilation.
2. **Add robust error handling** – Use a `ProducerCallback` to capture send failures, implement exponential back‑off retries, and integrate with the system’s logging/monitoring framework.