# Summary
`MOVEDAO.properties` defines the SQL INSERT statement (`insert.balance`) used by the Balance Notification batch job to persist each processed balance‑update event into the Hive table `mnaas.api_balance_update`. The DAO layer reads this property to execute a prepared statement with 16 parameters for every event.

# Key Components
- **insert.balance** – Property key containing a parameterised Hive INSERT with 16 placeholders (`?`).
- **DAO layer (BalanceNotfHandler)** – Reads `MOVEDAO.properties`, prepares the statement, binds event fields, and executes the insert against Hive/Impala.

# Data Flow
- **Input**: In‑memory `BalanceEvent` object (fields: initiator_id, transaction_id, source_id, tenancy_id, event_id, parent_event_id, entity_type, businessunit_id, event_class, event_time, businessunit, balance_event_name, msisdn, iccid, imsi, balance).
- **Processing**: DAO loads `insert.balance`, creates a `PreparedStatement`, binds the 16 fields in order, and executes.
- **Output**: Row inserted into Hive table `mnaas.api_balance_update`.
- **Side Effects**: Transaction log entry (via Log4j); possible Hive commit/rollback.
- **External Services**: Hive/Impala cluster (configured via `MNAAS_ShellScript.properties`), optional Kafka for downstream notifications.

# Integrations
- **MNAAS_ShellScript.properties** – Supplies Hive endpoint, authentication, and JDBC URL used by the DAO to obtain a connection.
- **log4j.properties** – Provides logging for DAO execution.
- **BalanceNotfHandler batch scripts** – Invoke the DAO during the ETL phase after extracting balance events from Oracle (via `ora_*` properties).

# Operational Risks
- **Schema drift** – Mismatch between property placeholders and Hive table schema can cause runtime errors. *Mitigation*: version‑controlled schema migrations and unit tests validating column count.
- **Connection failure** – Hive endpoint unavailable leads to batch failure. *Mitigation*: retry logic with exponential back‑off and alerting via email (configured in `MNAAS_ShellScript.properties`).
- **Data type mismatch** – Incorrect Java type binding may cause insert failures. *Mitigation*: strict POJO validation before DAO call.

# Usage
```bash
# Run the Balance Notification batch (shell wrapper reads properties)
cd move-mediation-bal-notification/BalanceNotfHandler
./runBalanceNotf.sh   # script loads MNAAS_ShellScript.properties, then Java job
# Debug DAO statement
java -cp target/balance-notf.jar com.mnaas.dao.BalanceDao -DlogLevel=DEBUG
```

# configuration
- **Environment variables**: `HIVE_JDBC_URL`, `HIVE_USER`, `HIVE_PASSWORD` (set by `MNAAS_ShellScript.properties`).
- **Config files**: `MOVEDAO.properties` (this file), `MNAAS_ShellScript.properties` (connection details), `log4j.properties` (logging).

# Improvements
1. Externalise column list into a separate property to decouple schema from SQL string, enabling easier schema evolution.
2. Add unit test harness that loads `MOVEDAO.properties` and validates that a mock `PreparedStatement` receives exactly 16 parameters in the correct order.