# Summary
`AggregationDataAccess` provides DAO operations for the Move Mediation Notification Handler. It retrieves SIM‑to‑MSISDN swap information from Hive tables and persists aggregated swap records into the `sim_msisdn_swap_aggr` table. All methods throw `DatabaseException` on failure and log activity via Log4j.

# Key Components
- **class AggregationDataAccess**
  - `getSIMSwapData(String oldSIM)`: fetches business unit, service, secsId, commercial offer, and old MSISDN for a given SIM.
  - `getMSISDNSwapData(String oldMSISDN)`: fetches business unit, service, secsId, commercial offer, and old SIM for a given MSISDN.
  - `insertInSwapTable(List<SwapRecord> insertRecords)`: inserts one or more `SwapRecord` rows into the aggregate swap table.
  - `getStackTrace(Exception)`: utility to convert an exception stack trace to a string for logging.
- **Dependencies**
  - `JDBCConnection`: wrapper that supplies Hive `Connection` objects via `getConnectionHive()`.
  - `ResourceBundle daoBundle`: loads SQL statements from `MOVEDAO.properties`.
  - `SwapRecord`: DTO holding swap‑related fields.
  - `DatabaseException`: custom unchecked exception for DAO errors.
  - Log4j `Logger` for operational/audit logging.

# Data Flow
| Method | Input | DB Interaction | Output | Side Effects |
|--------|-------|----------------|--------|--------------|
| `getSIMSwapData` | `oldSIM` (String) | Executes `get.sim.info` query on Hive; reads result set | `List<SwapRecord>` (populated with old MSISDN data) | Logs start, row count, and any errors |
| `getMSISDNSwapData` | `oldMSISDN` (String) | Executes `get.msisdn.info` query on Hive; reads result set | `List<SwapRecord>` (populated with old SIM data) | Logs start, row count, and any errors |
| `insertInSwapTable` | `List<SwapRecord>` | Executes `insert.swap` prepared statement per record on Hive | `void` (records persisted) | Logs batch size and completion; throws on failure |
| `getStackTrace` | `Exception` | None | `String` (stack trace) | None |

External services: Hive metastore/cluster accessed via JDBC; no message queues or REST endpoints are invoked directly.

# Integrations
- **SQL Definitions**: Keys `get.sim.info`, `get.msisdn.info`, `insert.swap` are defined in `MOVEDAO.properties` and referenced via `ResourceBundle`.
- **Constants**: Path to `MOVEDAO.properties` is resolved through `Constants` (runtime file path variables).
- **Logging**: Configured by `log4j.properties`; logs are emitted to daily rolling file and STDOUT.
- **Upstream**: Called by service layer components that orchestrate SIM‑MSISDN swap processing (e.g., Kafka consumer handling swap events).
- **Downstream**: Inserted rows are later consumed by analytics or reporting jobs that read `sim_msisdn_swap_aggr`.

# Operational Risks
- **Resource Leak**: `ResultSet` closed after `Connection` and `Statement`; however order is reversed (should close `ResultSet` first). Mitigation: use try‑with‑resources.
- **SQL Injection**: Prepared statements are used correctly; risk minimal.
- **Hive Connection Saturation**: Each DAO call opens a new connection; high concurrency may exhaust Hive connections. Mitigation: implement connection pooling.
- **Partial Batch Failure**: `insertInSwapTable` executes each record individually; a failure aborts remaining inserts without rollback. Mitigation: batch execution with transaction handling or error collection.
- **Logging Sensitive Data**: SIM/MSISDN values are logged at INFO level. Mitigation: mask or downgrade to DEBUG in production.

# Usage
```java
AggregationDataAccess dao = new AggregationDataAccess();

// Debug fetch by SIM
List<SwapRecord> simData = dao.getSIMSwapData("123456789012345");

// Debug fetch by MSISDN
List<SwapRecord> msisdnData = dao.getMSISDNSwapData("919876543210");

// Insert aggregated records
dao.insertInSwapTable(simData);
```
Run within the application container where `MOVEDAO.properties` and `log4j.properties` are on the classpath. For unit testing, mock `JDBCConnection` and `ResourceBundle`.

# Configuration
- **MOVEDAO.properties** (on classpath): contains SQL statements referenced by keys `get.sim.info`, `get.msisdn.info`, `insert.swap`.
- **log4j.properties**: defines `${logfile.name}` for daily rolling file appender.
- **Environment**: Hive JDBC URL, username/password, and any Kerberos/SSL settings are read inside `JDBCConnection` (not shown here). Ensure those values are set in the runtime environment or system properties.

# Improvements
1. Refactor all DAO methods to use Java 7+ try‑with‑resources for deterministic resource cleanup and correct close order.
2. Convert `insertInSwapTable` to batch execution (`addBatch`/`executeBatch`) with a single transaction to improve throughput and provide atomicity.