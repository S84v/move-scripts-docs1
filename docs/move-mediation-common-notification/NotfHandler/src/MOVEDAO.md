# Summary  
`MOVEDAO.properties` is the DAO‑layer configuration for the Move Mediation Notification Handler. It defines all named SQL statements (INSERT, UPDATE, SELECT) used at runtime to persist raw notification events, location data, SIM/IMEI updates, OCS bypass records, eSIM notifications and aggregation records into the **mnaas** schema (Oracle/Impala). The DAO reads these keys, builds `PreparedStatement`s, binds the positional parameters supplied by the service layer, and executes them against the Move database.

# Key Components  
- **insert.raw** – Inserts a full notification payload into `mnaas.api_notification_raw` (partitioned by `event_date`).  
- **insert.raw.reject** – Inserts rejected notifications into `api_notification_raw_fail`.  
- **insert.location** – Persists enriched location events into `mnaas.api_notification_location` (partitioned).  
- **insert.location.attr** – Stores custom attribute key/value pairs for a location event.  
- **insert.location.raw** – Stores the raw location payload into `mnaas.api_notification_location_raw`.  
- **insert.simimei** – Persists SIM‑IMEI change events into `mnaas.sim_imei_notification`.  
- **insert.ocs** – Persists OCS bypass data into `mnaas.ocs_bypass_data`.  
- **insert.esim** – Persists eSIM provisioning notifications into `mnaas.api_notification_esim`.  
- **get.sim.info / get.msisdn.info** – Lookup queries against `mnaas.actives_raw_daily` to resolve SIM ↔ MSISDN relationships.  
- **insert.swap** – Inserts a row into the aggregation table `mnaas.sim_msisdn_swap_aggr`.  
- **notif.status.update / notif.status.update.preact** – Updates SIM inventory status in `MOVE_SIM_INVENTORY_STATUS`.  
- **notif.get.imsi** – Retrieves all IMSI values for a SIM.  
- **notif.add.imsi** – Writes IMSI values back to the inventory table (by `PRODUCT_ID`).  
- **notif.imei.update** – Updates IMEI, source, and timestamp for a SIM.

# Data Flow  
| Stage | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| Service layer → DAO | Java objects containing event fields (e.g., initiatorId, transactionId, msisdn, iccid, etc.) | DAO reads the matching property key, creates a `PreparedStatement`, binds parameters in the order defined, executes | Row inserted/updated in the target table (Oracle/Impala). |
| Validation / Rejection | Event fails business validation | DAO uses `insert.raw.reject` | Row inserted into `api_notification_raw_fail` with reason code/reason. |
| Lookup | SIM or MSISDN value | DAO executes `get.sim.info` or `get.msisdn.info` | Result set containing business unit, service abbreviation, TCL SECS ID, commercial offer, counterpart identifier. |
| Aggregation | Completed swap event | DAO executes `insert.swap` | Row added to `sim_msisdn_swap_aggr` (partitioned). |
| Inventory update | SIM status change request | DAO executes `notif.status.update*` and optional IMSI/IMEI updates | `MOVE_SIM_INVENTORY_STATUS` rows modified (status, timestamps, availability). |

External services touched indirectly: Oracle database (primary), Impala/Hive (for partitioned tables), Kafka (notification ingestion – not in this file), SMTP (audit mail – not in this file).

# Integrations  
- **DAO Implementation** – `MOVEDAO.java` (or equivalent) loads this properties file via `java.util.Properties`, maps keys to `PreparedStatement`s.  
- **MNAAS Shell Script (`MNAAS_ShellScript.properties`)** – Supplies JDBC URL, user, password, and connection pool settings used by the DAO at runtime.  
- **Notification Ingestion Pipeline** – Kafka consumer reads raw events, transforms them into POJOs, and invokes DAO methods that reference the statements defined here.  
- **Log4j (`log4j.properties`)** – Provides logging for DAO execution (SQL statements, execution time, errors).  
- **Batch/ETL Jobs** – May invoke the same DAO for bulk load or reconciliation tasks.

# Operational Risks  
- **Schema drift** – Adding/removing columns in target tables without updating the corresponding property entry will cause `SQLException` (parameter count mismatch). *Mitigation*: automated schema‑to‑property validation CI step.  
- **Partition key mismatch** – `event_date`/`partition_date` must be supplied correctly; missing or malformed dates cause insert failures. *Mitigation*: enforce date formatting in service layer and add unit tests.  
- **Hard‑coded column order** – Positional binding is fragile; any reorder requires property update. *Mitigation*: adopt named‑parameter framework or generate statements from metadata.  
- **Bulk rejection handling** – High volume of rejects may fill `api_notification_raw_fail`. *Mitigation*: monitor table size, implement archiving/TTL.  
- **Connection leakage** – DAO must close `PreparedStatement`/`ResultSet` in finally blocks. *Mitigation*: use try‑with‑resources or a connection pool with leak detection.

# Usage  
```java
// Example DAO usage (debugging)
Properties sqlProps = new Properties();
sqlProps.load(new FileInputStream("MOVEDAO.properties"));

String sql = sqlProps.getProperty("insert.raw");
try (Connection con = dataSource.getConnection();
     PreparedStatement ps = con.prepareStatement(sql)) {
    // bind 35 parameters in order
    ps.setString(1, initiatorId);
    // ...
    ps.setString(35, isSbsMember);
    int rows = ps.executeUpdate();
    System.out.println("Inserted rows: " + rows);
}
```
To debug a specific statement, enable `log4j.logger.com.move.dao=DEBUG` to log the final SQL with bound values.

# Configuration  
- **Environment variables** (provided by `MNAAS_ShellScript.properties`):  
  - `ORA_SERVERNAME_MOVE`, `ORA_PORT`, `ORA_SID` – Oracle connection details.  
  - `DB_USER`, `DB_PASSWORD` – Credentials.  
  - `IMPALA_ENDPOINT`, `HIVE_ENDPOINT` – For partitioned tables if using Impala.  
- **Referenced