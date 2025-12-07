# Summary
`Constants.java` is a central repository of immutable string literals and mutable path variables used throughout the Move Mediation Notification Handler (MNAAS). It defines the canonical names for consumer‑type identifiers, Kafka topic names, error‑table identifiers, and holds runtime‑set file‑path references (`logFile`, `propertyFile`). In production the class is loaded once at JVM startup and its fields are referenced by DAO, producer, consumer, and logging components to ensure consistent naming across the notification ingestion pipeline.

# Key Components
- **`public class Constants`**
  - **Consumer‑type literals** – e.g., `SIM_SWAP`, `MSISDN_SWAP`, `CHANGE_STATUS`, etc. (used to map incoming API events to internal processing logic).
  - **Kafka topic literals** – e.g., `TOPIC_SIM_SWAP`, `TOPIC_MSISDN_SWAP`, `TOPIC_E_SIM`, etc. (used by Kafka producers/consumers).
  - **Error‑table identifiers** – `ERROR_RAW`, `ERROR_SWAP`, `ERROR_FAIL` (used when routing failed records to specific DB tables).
  - **Mutable path holders** – `public static String logFile` and `propertyFile` (populated at runtime from external property files).

# Data Flow
| Element | Input | Output | Side‑Effect / External Interaction |
|---------|-------|--------|------------------------------------|
| `Constants` fields | None (static initialization) | Provide constant strings to any class that imports `com.tcl.move.constants.Constants` | None |
| `logFile` / `propertyFile` | Set by `MNAAS_ShellScript.properties` or programmatic bootstrap code | Accessible to logging subsystem and configuration loaders | Determines file locations for Log4j output and property file reads |
| Consumer‑type literals | Used by event‑parsing logic (e.g., REST controllers) | Drive routing to DAO methods | Influence which SQL statements from `MOVEDAO.properties` are executed |
| Topic literals | Used by Kafka producer/consumer wrappers | Publish/subscribe to specific Kafka topics | Direct interaction with Kafka brokers defined in `MNAAS_ShellScript.properties` |

# Integrations
- **DAO Layer (`MOVEDAO.properties`)** – References consumer‑type constants to select appropriate SQL statements.
- **Kafka Producer/Consumer Modules** – Use topic constants to configure `KafkaProducer` and `KafkaConsumer` instances.
- **Logging (`log4j.properties`)** – Reads `Constants.logFile` to resolve `${logfile.name}` placeholder.
- **Bootstrap/Init Scripts** – External shell scripts or Java bootstrap code assign values to `logFile` and `propertyFile` based on environment‑specific property files (`MNAAS_ShellScript.properties`).

# Operational Risks
- **Hard‑coded strings**: Any change to external API endpoints or topic names requires code recompilation and redeployment.
- **Mutable static path variables**: Not thread‑safe; concurrent updates can lead to inconsistent logging or property loading.
- **Missing validation**: If `logFile`/`propertyFile` remain `null`, downstream components may throw `NullPointerException`.
- **Duplication risk**: Topic names duplicated in property files may drift, causing mismatched producer/consumer configurations.

# Usage
```java
// Example: publishing a SIM‑Swap notification
String topic = Constants.TOPIC_SIM_SWAP;
ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, payload);
kafkaProducer.send(record);

// Example: initializing file paths (executed once during startup)
Constants.logFile = System.getProperty("log.file.path");
Constants.propertyFile = System.getenv("MNAAS_PROP_FILE");
```
To debug, attach a breakpoint on any reference to `Constants` and verify that the static fields contain expected values after bootstrap.

# Configuration
- **Environment Variables / System Properties**
  - `log.file.path` – optional system property to set `Constants.logFile`.
  - `MNAAS_PROP_FILE` – optional environment variable to set `Constants.propertyFile`.
- **External Property Files**
  - `MNAAS_ShellScript.properties` – supplies the actual file paths used by the bootstrap code that populates the mutable fields.
  - `log4j.properties` – references `${logfile.name}` which resolves to `Constants.logFile`.

# Improvements
1. **Externalize all literals**: Move consumer‑type strings, topic names, and error identifiers to a dedicated properties file (e.g., `constants.properties`) and load them at startup. This eliminates recompilation for naming changes.
2. **Replace mutable static paths with immutable configuration objects**: Introduce a `Configuration` POJO populated once via a builder or dependency‑injection framework (Spring, Guice). Provide thread‑safe accessors and deprecate the public mutable fields.