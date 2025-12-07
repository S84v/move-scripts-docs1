# Summary
`MOVEDAO.properties` provides parameterised SQL INSERT statements for the AsyncNotfHandler DAO layer. In production the DAO loads these statements to persist asynchronous notification events, detailed records, custom attributes, and addon information into the `mnaas` schema of the Oracle database, enabling downstream processing and audit trails for the telecom move system.

# Key Components
- **AsyncNotfDAO** – Java DAO class that reads `MOVEDAO.properties` and executes the prepared statements.  
- **insert.async** – Inserts a row into `mnaas.api_notification_async`.  
- **insert.async.det** – Inserts a row into `mnaas.api_notification_async_det`.  
- **insert.async.attr** – Inserts a row into `mnaas.api_notification_async_attr`.  
- **insert.addon** – Inserts a row into partitioned table `mnaas.api_notification_addon` (partitioned by `event_date`).  

# Data Flow
- **Input:** Java domain objects representing a notification event (initiator, transaction IDs, entity details, etc.).  
- **Processing:** DAO maps object fields to the `?` placeholders of the prepared statements loaded from this file.  
- **Output:** Rows written to the Oracle tables listed above.  
- **Side Effects:** Transaction commit/rollback; audit logs via Log4j.  
- **External Services/DBs:** Oracle DB instance defined in `MNAAS_ShellScript.properties`; optional downstream consumers (Kafka, Hive) read the persisted rows.

# Integrations
- **MNAAS_ShellScript.properties** – Supplies DB connection URL, credentials, and driver class used by the DAO at runtime.  
- **AsyncNotfHandler** service – Calls AsyncNotfDAO methods to persist events after business logic execution.  
- **Log4j** – Logs success/failure of each insert operation.  
- **Kafka/Hive** – Not directly referenced in this file but downstream components consume the persisted data.

# Operational Risks
- **Schema drift:** Table column changes break the positional mapping of `?` placeholders. *Mitigation:* version‑controlled schema migrations and unit tests that validate statement parameter counts.  
- **Partition mis‑routing:** `insert.addon` relies on `event_date` partition; incorrect or missing date leads to insert failures. *Mitigation:* enforce non‑null `event_date` in the domain model and add validation before DAO call.  
- **Connection exhaustion:** High volume of async inserts may saturate DB connections. *Mitigation:* use a connection pool with appropriate max size and monitor pool metrics.  
- **Data loss on failure:** If a transaction rolls back, downstream systems may miss events. *Mitigation:* implement retry logic and dead‑letter queue for failed inserts.

# Usage
```java
// Load properties
Properties sqlProps = new Properties();
try (InputStream is = getClass().getResourceAsStream("/MOVEDAO.properties")) {
    sqlProps.load(is);
}

// Prepare DAO
AsyncNotfDAO dao = new AsyncNotfDAO(dataSource, sqlProps);

// Example insert
NotificationEvent evt = new NotificationEvent(...);
dao.insertAsync(evt);   // uses sqlProps.getProperty("insert.async")
```
To debug, enable `log4j.logger.com.company.AsyncNotfDAO=DEBUG` to see bound parameter values.

# configuration
- **Environment Variables:** `DB_HOST`, `DB_PORT`, `DB_SERVICE`, `DB_USER`, `DB_PASSWORD` (populated from `MNAAS_ShellScript.properties`).  
- **Config Files:** `MOVEDAO.properties` (SQL statements), `MNAAS_ShellScript.properties` (DB connection), `log4j.properties` (logging).  

# Improvements
1. **Externalise partition key:** Replace hard‑coded `partition (event_date)` with a configurable partition expression to support future partitioning strategies.  
2. **Batch insert support:** Add prepared statements for bulk inserts to reduce round‑trips and improve throughput under high load.