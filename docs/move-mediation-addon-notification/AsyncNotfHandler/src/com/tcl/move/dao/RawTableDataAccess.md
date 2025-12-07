# Summary
`RawTableDataAccess` is a DAO class that persists a batch of `AddonRecord` objects into the Hive/Impala raw table `mnaas.api_notification_addon`. It retrieves the INSERT statement from `MOVEDAO.properties`, obtains a Hive JDBC connection via `JDBCConnection`, executes a prepared‑statement batch, and logs/throws `DatabaseException` on failure.  

# Key Components
- **class RawTableDataAccess** – encapsulates raw‑table write logic.  
- **ResourceBundle daoBundle** – loads SQL from `MOVEDAO.properties`.  
- **JDBCConnection jdbcConn** – factory for Hive (`getConnectionHive`) connections.  
- **public void insertInAddonRawTable(List<AddonRecord>)** – validates input, prepares statement, maps 33 fields, executes batch, handles cleanup.  
- **private String getStackTrace(Exception)** – utility to convert stack trace to string for logging.  

# Data Flow
| Stage | Source | Destination | Description |
|-------|--------|-------------|-------------|
| Input | Caller (service layer) | `insertInAddonRawTable` | `List<AddonRecord>` containing 33 fields per record. |
| Load | `MOVEDAO.properties` (classpath) | `daoBundle` | Retrieves key `insert.addon` → parameterised INSERT SQL. |
| Connection | Hive/Impala cluster | `Connection` via `jdbcConn.getConnectionHive()` | JDBC URL built from system properties (driver, host, port). |
| Write | PreparedStatement (batch) | Hive table `mnaas.api_notification_addon` (partitioned by `event_date`) | Each record maps to positional parameters 1‑33, then `addBatch()`. |
| Execution | JDBC driver | Hive | `statement.executeBatch()` persists rows. |
| Side Effects | DB | None external | Logs via Log4j, may raise `DatabaseException`. |
| Output | None (void) | – | Success: rows inserted; Failure: exception propagated. |

# Integrations
- **Constants** – provides error code `Constants.ERROR_RAW` used in exception construction.  
- **JDBCConnection** – shared connection factory used by other DAO classes (e.g., `AsyncNotfDAO`).  
- **AddonRecord DTO** – data model populated by upstream services that parse incoming Kafka messages (`TOPIC_ASYNC`).  
- **MOVEDAO.properties** – centralised SQL repository consumed by multiple DAO implementations.  

# Operational Risks
- **Batch size overflow** – Very large `insertRecords` may exceed Hive driver limits → mitigate by chunking batches (e.g., 5 000 rows).  
- **Resource leak** – Failure before `finally` may leave open connections; current finally block closes but re‑throws on close error – consider suppressing close exceptions.  
- **SQL injection** – All fields are set via `setString`; however, numeric fields are converted to string (`amount + ""`) – ensure DB schema accepts string or cast appropriately.  
- **Schema drift** – If `MOVEDAO.properties` INSERT column order diverges from `AddonRecord` fields, data misalignment occurs – enforce version control and integration tests.  

# Usage
```java
// Unit test / debugging example
List<AddonRecord> records = new ArrayList<>();
records.add(new AddonRecord(/* populate all 33 fields */));

RawTableDataAccess dao = new RawTableDataAccess();
try {
    dao.insertInAddonRawTable(records);
    System.out.println("Insert successful");
} catch (DatabaseException e) {
    e.printStackTrace();
}
```
Run with JVM system properties for Hive connection (e.g., `-Dhive.host=... -Dhive.port=...`).  

# configuration
- **System properties** (read by `JDBCConnection`): `hive.driver`, `hive.host`, `hive.port`, `hive.user`, `hive.password`.  
- **Classpath resource**: `MOVEDAO.properties` containing key `insert.addon` with the full INSERT statement.  
- **Log4j configuration**: `log4j.properties` referenced via `Constants` for log file locations.  

# Improvements
1. **Batch size management** – Add logic to split `insertRecords` into configurable batch chunks to avoid driver limits and memory pressure.  
2. **Typed parameter binding** – Replace string conversion for numeric fields (`amount`) with appropriate `setBigDecimal`/`setInt` to match DB column types and improve performance.  