# Summary
`Constants.java` defines immutable string literals and mutable path variables used across the AsyncNotfHandler component. It centralizes identifiers for consumer type, Kafka topic, error categorization, and runtime file locations (log and property files) that are referenced by DAO, service, and logging modules during production move events.

# Key Components
- **Constants class**
  - `ASYNC` – identifier for async callback notifications.
  - `TOPIC_ASYNC` – Kafka topic name for async callback events.
  - `ERROR_RAW` – label for raw‑table error classification.
  - `logFile` – mutable holder for the absolute path of the runtime log file.
  - `propertyFile` – mutable holder for the absolute path of the DAO/property configuration file.

# Data Flow
- **Inputs**: Values assigned to `logFile` and `propertyFile` at application start (e.g., from command‑line args or environment variables).
- **Outputs**: Constants are read by:
  - Kafka producer/consumer modules to publish/subscribe to `TOPIC_ASYNC`.
  - DAO layer to locate `MOVEDAO.properties` via `propertyFile`.
  - Log4j initialization to resolve `${logfile.name}` via `logFile`.
- **Side Effects**: None; class only provides static data.
- **External Services**: Kafka broker (topic name), Oracle DB (via DAO), file system (log/property files).

# Integrations
- **AsyncNotfHandler**: Consumes `Constants.ASYNC` to tag notification payloads.
- **Kafka Producer**: Uses `Constants.TOPIC_ASYNC` when publishing async callback messages.
- **DAO Layer**: Reads `Constants.propertyFile` to load SQL statements from `MOVEDAO.properties`.
- **Log4j**: References `Constants.logFile` through `${logfile.name}` placeholder in `log4j.properties`.

# Operational Risks
- **Null path variables**: `logFile` and `propertyFile` default to `null`; if not set before use, `NullPointerException` will occur.
- **Hard‑coded strings**: Changing topic or error label requires code recompilation.
- **Visibility**: Mutable static fields are not thread‑safe; concurrent updates may cause race conditions.

# Usage
```java
// Application bootstrap
Constants.logFile = System.getenv("MNAAS_LOG_PATH");
Constants.propertyFile = System.getenv("MNAAS_PROP_PATH");

// Example usage in producer
producer.send(new ProducerRecord<>(Constants.TOPIC_ASYNC, key, value));
```
Debug by inspecting `Constants.logFile` and `Constants.propertyFile` values after initialization.

# configuration
- Environment variables or startup parameters must set:
  - `MNAAS_LOG_PATH` → absolute path for log file.
  - `MNAAS_PROP_PATH` → absolute path for DAO/property file.
- Referenced config files:
  - `log4j.properties` (uses `${logfile.name}`).
  - `MOVEDAO.properties` (SQL statements).

# Improvements
1. Convert `logFile` and `propertyFile` to `final` after initialization and expose via getter to enforce immutability.
2. Externalize `ASYNC`, `TOPIC_ASYNC`, and `ERROR_RAW` to a properties file (e.g., `MNAAS_ShellScript.properties`) to allow runtime reconfiguration without code changes.