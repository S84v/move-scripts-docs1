# Summary
`MOVEDAO.properties` is a DAO‑layer configuration file for the **Move Mediation Notification Handler**. It defines named SQL statements (INSERT, UPDATE, SELECT) used by the notification service to persist raw notification events, location data, SIM/IMEI updates, OCS bypass records, eSIM notifications and related aggregation tables in the **mnaas** schema. The statements are referenced by the Java DAO implementation at runtime to build `PreparedStatement`s, bind parameters, and execute against the production Hive/Impala (or compatible) data warehouse.

# Key Components
- **insert.raw** – Insert raw API notification events (partitioned by `event_date`).  
- **insert.raw.reject** – Insert failed‑notification records into `api_notification_raw_fail`.  
- **insert.location** – Insert enriched location‑change events (partitioned by `event_date`).  
- **insert.location.attr** – Insert custom attribute key/value pairs for a location event.  
- **insert.location.raw** – Insert low‑level location raw payload (partitioned by `event_date`).  
- **insert.simimei** – Insert SIM‑IMEI change notifications (partitioned by `event_date`).  
- **insert.ocs** – Insert OCS bypass data (partitioned by `event_date`).  
- **insert.esim** – Insert eSIM provisioning notifications (non‑partitioned).  
- **get.sim.info / get.msisdn.info** – Lookup active SIM/MSISDN details from `actives_raw_daily`.  
- **insert.swap** – Insert SIM↔MSISDN swap aggregation rows (partitioned by `partition_date`).  
- **notif.status.update / notif.status.update.preact** – Update SIM inventory status and availability flag.  
- **notif.get.imsi** – Retrieve up‑to‑10 IMSI values for a SIM.  
- **notif.add.imsi** – Bulk‑update IMSI columns for a product record.  
- **notif.imei.update** – Update IMEI, source, and timestamp for a SIM.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | Notification DTO (raw, location, SIM‑IMEI, OCS, eSIM) | DAO loads matching SQL from this file, creates `PreparedStatement`, binds positional parameters (`?`). | Row inserted into target table (partitioned where applicable). |
| 2 | SIM identifier | `notif.get.imsi` SELECT → ResultSet of IMSI values. | Returned IMSI list for downstream enrichment. |
| 3 | SIM identifier + status fields | `notif.status.update*` UPDATE → modifies `MOVE_SIM_INVENTORY_STATUS`. | Updated inventory status, `SIM_AVAILABLE='N'`. |
| 4 | MSISDN/SIM lookup request | `get.sim.info` / `get.msisdn.info` SELECT → latest active record. | Business unit, service abbreviation, commercial offer, etc. |
| 5 | Swap event DTO | `insert.swap` INSERT → aggregation table. | Row for reporting/analytics. |
| 6 | Failure handling | `insert.raw.reject` INSERT on exception. | Failure audit record. |

All statements assume a Hive/Impala‑compatible JDBC driver; partition clauses (`partition (event_date)`) rely on the target table being pre‑partitioned by the `event_date` column.

# Integrations
- **Java DAO Layer** (`com.tcl.move.notification.dao.*`) loads this properties file via `PropertiesLoader` and maps each key to a `String` SQL template.
- **Notification Service** (`NotificationHandler`) builds DTOs from incoming events (Kafka, REST, etc.) and invokes DAO methods.
- **Hive/Impala Cluster** – target for all DML statements; requires Kerberos/LDAP authentication configured externally.
- **Logging** – `log4j.properties` (see sibling config) records execution success/failure.
- **Batch Scheduler** – nightly jobs may invoke the DAO for bulk swap aggregation (`insert.swap`).

# Operational Risks
- **Partition Mismatch** – If `event_date`/`partition_date` values do not align with existing Hive partitions, inserts will fail or create empty partitions, leading to data loss. *Mitigation*: Validate date fields before DAO call; pre‑create daily partitions via Hive metastore scripts.
- **Schema Drift** – Adding/removing columns in target tables without updating this file will cause `Invalid column` errors. *Mitigation*: Version‑control DAO properties with schema migration scripts; run CI checks.
- **Prepared‑Statement Parameter Order** – The `?` placeholders must match the exact order expected by the caller; any mismatch yields data corruption. *Mitigation*: Unit‑test each DAO method with mock parameters; enforce naming conventions.
- **Large Batch Inserts** – Bulk inserts without proper `INSERT … VALUES (…)` batching can overwhelm Hive metastore. *Mitigation*: Use `PreparedStatement.addBatch()` and appropriate `executeBatch()` size (e.g., 500 rows).
- **Security** – No credential handling in this file, but exposure of raw SQL may aid injection if callers concatenate strings. *Mitigation*: Enforce strict use of `PreparedStatement`; reject any dynamic SQL construction.

# Usage
```java
// Example: persisting a raw notification
ExportRecord rec = ...; // populated from inbound event
Properties daoProps = new Properties();
daoProps.load(new FileInputStream("MOVEDAO.properties"));
String sql = daoProps.getProperty("insert.raw");

// Obtain JDBC connection (Hive)
try (Connection conn = HiveConnectionProvider.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {

    ps.setString(1, rec.getInitiatorId());
    ps.setString(2, rec.getTransactionId());
    // ... set remaining 34 parameters in order ...
    ps.setString(35, rec.getIsSbsMember());

    ps.executeUpdate();
}
```
*Debugging*: Enable `log4j.logger.com.tcl.move.notification=DEBUG` to log the final bound SQL (with parameter values) via a custom `PreparedStatement` wrapper.

# Configuration
- **Environment Variables / System Properties**
  - `hive.metastore.uris` – Hive metastore endpoint.
  - `hive.server2.jdbc.url` – JDBC URL for Hive/Impala.
  - `logfile.name` – Used by `log4j