# Summary
`Constants.java` defines a static repository of string literals and mutable file‑path variables used throughout the **BalanceNotfHandler** batch job. The constants provide identifiers for the balance‑update event type, the Kafka topic to which balance notifications are published, an error‑category label, and placeholders for runtime‑determined log and property file locations.

# Key Components
- **Class `com.tcl.move.constants.Constants`**
  - `public static final String BALANCE` – human‑readable name of the balance‑update event.
  - `public static final String TOPIC_BALANCE` – Kafka topic name for balance‑update notifications.
  - `public static final String ERROR_RAW` – label used when persisting raw table errors.
  - `public static String logFile` – mutable reference to the active log file path (set at runtime).
  - `public static String propertyFile` – mutable reference to the active properties file path (set at runtime).

# Data Flow
- **Inputs:** Values for `logFile` and `propertyFile` are injected by the batch launcher (e.g., from command‑line arguments or environment variables).
- **Outputs:** No direct output; constants are read by other classes (DAO, Kafka producer, logger configuration) to construct messages, select Kafka topics, and locate configuration files.
- **Side Effects:** Assignment to `logFile`/`propertyFile` modifies global state visible to all threads in the JVM.
- **External Services/DBs/Queues:** Indirectly influences interactions with:
  - Kafka broker (via `TOPIC_BALANCE`).
  - Oracle/Impala/Hive (via property files referenced by `propertyFile`).
  - Log4j (via `logFile` used in `log4j.properties`).

# Integrations
- **Log4j configuration** (`log4j.properties`) reads `${logfile.name}` which is set from `Constants.logFile`.
- **Shell script launcher** (`MNAAS_ShellScript.properties`) supplies the path to the properties file; the Java entry point assigns it to `Constants.propertyFile`.
- **Kafka producer** in the BalanceNotfHandler code references `Constants.TOPIC_BALANCE` when publishing balance‑update events.
- **Error handling** modules compare error categories against `Constants.ERROR_RAW`.

# Operational Risks
- **Mutable global state:** Concurrent modifications to `logFile`/`propertyFile` can cause race conditions in multi‑threaded runs. *Mitigation:* set values once during JVM startup; make fields `final` after initialization.
- **Hard‑coded topic name:** Changing the Kafka topic requires code redeployment. *Mitigation:* externalize `TOPIC_BALANCE` to a properties file.
- **Missing initialization:** If `logFile` or `propertyFile` remain `null`, downstream components may throw `NullPointerException`. *Mitigation:* add validation at startup and fail fast with clear error messages.

# Usage
```bash
# Example launch (wrapper script sets env vars)
export LOG_FILE=/var/log/move/balance_notf.log
export PROP_FILE=/opt/move/config/MNAAS_ShellScript.properties

java -cp balance-notf-handler.jar \
     -Dlogfile.name=$LOG_FILE \
     -Dproperty.file=$PROP_FILE \
     com.tcl.move.balance.BalanceNotfHandlerMain
```
Within `BalanceNotfHandlerMain`:
```java
Constants.logFile = System.getProperty("logfile.name");
Constants.propertyFile = System.getProperty("property.file");
```

# configuration
- **Environment variables / system properties**
  - `logfile.name` → assigned to `Constants.logFile`
  - `property.file` → assigned to `Constants.propertyFile`
- **Referenced config files**
  - `log4j.properties` (uses `${logfile.name}`)
  - `MNAAS_ShellScript.properties` (provides DB/Kafka credentials)

# Improvements
1. Convert `logFile` and `propertyFile` to immutable `final` fields initialized via a static factory method to prevent accidental reassignment.
2. Externalize `TOPIC_BALANCE`, `BALANCE`, and `ERROR_RAW` into a dedicated properties file to enable runtime reconfiguration without code changes.