# Summary
`MOVEDAO.properties` defines parameterised SQL INSERT statements used by the **AsyncNotfHandler** DAO layer to persist asynchronous notification events, detailed records, custom attributes, and addon information into the `mnaas` schema. In production the file drives write‑through of event data to the relational database, supporting downstream processing and audit trails for the telecom move system.

# Key Components
- `insert.async` – inserts a row into `mnaas.api_notification_async`.
- `insert.async.det` – inserts a row into `mnaas.api_notification_async_det`.
- `insert.async.attr` – inserts a row into `mnaas.api_notification_async_attr`.
- `insert.addon` – inserts a row into partitioned table `mnaas.api_notification_addon` (partitioned by `event_date`).

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| DAO load | Java code reads `MOVEDAO.properties` via `Properties` class | Statements are cached as `String` | PreparedStatement objects created at runtime | None |
| Execution | Method parameters (≈23‑35 values) supplied by `AsyncNotfHandler` | `PreparedStatement#setX` binds values; `executeUpdate()` runs SQL | Row(s) inserted into target tables | Transaction commit/rollback, DB write‑ahead logs |
| Partition handling | `insert.addon` includes `partition (event_date)` clause | DB routes row to appropriate daily partition | Data stored in correct partition | Partition creation/maintenance required |

# Integrations
- **AsyncNotfHandler** component reads this file to obtain SQL for `NotificationDAO`.
- **DataSource** (JDBC pool) provides DB connections; statements executed within the service’s transaction manager.
- **Log4j** (configured in `log4j.properties`) logs execution outcomes.
- **External services**: upstream event generators supply the parameter values; downstream consumers read from the `mnaas` tables.

# Operational Risks
- **SQL injection** – mitigated by using `PreparedStatement` with positional parameters.
- **Missing partition** – `insert.addon` will fail if the `event_date` partition does not exist; requires partition management job.
- **Schema drift** – changes to table columns without updating property placeholders will cause `SQLException`.
- **Batch size limits** – large payloads may exceed DB transaction limits; monitor and tune batch size.
- **Connection leaks** – ensure DAO closes `PreparedStatement`/`Connection` in finally block.

# Usage
```java
// Example snippet in NotificationDAO
Properties sqlProps = new Properties();
try (InputStream is = getClass().getResourceAsStream("/MOVEDAO.properties")) {
    sqlProps.load(is);
}
String sql = sqlProps.getProperty("insert.async");
try (Connection con = dataSource.getConnection();
     PreparedStatement ps = con.prepareStatement(sql)) {
    ps.setString(1, initiatorId);
    // ... set remaining parameters ...
    ps.executeUpdate();
}
```
Debug by enabling `DEBUG` level for the DAO logger in `log4j.properties` to print bound parameter values.

# configuration
- **Environment variables**: `DB_URL`, `DB_USER`, `DB_PASSWORD` (used by the DataSource).
- **Config files referenced**: `MOVEDAO.properties` (this file), `log4j.properties` (logging), `datasource.xml` or Spring/YAML DB config.
- **JDBC driver** must be compatible with the target RDBMS (e.g., Oracle, PostgreSQL).

# Improvements
1. **Externalize partition key** – replace hard‑coded `partition (event_date)` with a configurable placeholder to support dynamic partition strategies.
2. **Add checksum column** – include a hash of the inserted payload to enable idempotency checks and simplify duplicate detection.