# Summary
`MOVEDAO.properties` defines the SQL statement used by the Balance Notification batch job to persist balance update events into the Hive table `mnaas.api_balance_update`. The property `insert.balance` is read by DAO components to execute a parameterised INSERT for each processed balance event.

# Key Components
- **`insert.balance`** – Property key containing a prepared‑statement style INSERT with 16 placeholders (`?`).  
  - Columns: `initiator_id, transaction_id, source_id, tenancy_id, event_id, parent_event_id, entity_type, businessunit_id, event_class, event_time, businessunit, balance_event_name, msisdn, iccid, imsi, balance`.

# Data Flow
| Stage | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| DAO load | Java object representing a balance event (16 fields) | DAO reads `insert.balance` from `MOVEDAO.properties`, creates a `PreparedStatement`, binds fields to placeholders | Row inserted into Hive table `mnaas.api_balance_update` (persistent storage) |
| Error handling | SQL exceptions | Propagated to caller, logged via Log4j | Potential retry or job failure |

External services: Hive metastore / HiveServer2 accessed via JDBC/ODBC driver.

# Integrations
- **`SubsCountLoader` / other batch jobs** – Not directly referenced but shares the same `mnaas` schema.  
- **DAO layer** – `BalanceDAO` (or similarly named class) loads this properties file at initialization and uses the `insert.balance` statement for each event.  
- **Log4j configuration** – Logging of successful inserts or failures is governed by `log4j.properties` in the same module.  

# Operational Risks
- **Schema drift** – Adding/removing columns in `api_balance_update` without updating the property leads to runtime SQL errors. *Mitigation*: schema version check during job start‑up.  
- **Data type mismatch** – Incorrect Java type binding (e.g., passing a string for a numeric column) causes insert failures. *Mitigation*: enforce strict POJO typing and unit tests.  
- **Bulk load performance** – Single‑row inserts can degrade throughput under high event volume. *Mitigation*: batch the `PreparedStatement` or switch to Hive `INSERT INTO … SELECT` with staging files.  
- **Missing property file** – Job fails to start if `MOVEDAO.properties` is not on the classpath. *Mitigation*: CI validation of resource packaging.

# Usage
```bash
# Run the Balance Notification job (example wrapper script)
java -cp "BalanceNotfHandler.jar:conf/*" com.tcl.move.main.BalanceNotfHandler \
     -Dlogfile.name=/var/log/balance_notf.log \
     -Dconfig.file=conf/MOVEDAO.properties
```
During debugging, set `log4j.rootLogger=DEBUG` to view the generated SQL and bound parameters.

# configuration
- **Environment variables / system properties**  
  - `logfile.name` – referenced by `log4j.properties`.  
  - `config.file` – path to `MOVEDAO.properties` (if not on default classpath).  
- **Config files**  
  - `MOVEDAO.properties` – contains the `insert.balance` statement.  
  - `log4j.properties` – logging configuration.  

# Improvements
1. **Parameterisation via a DAO framework** – Replace raw property‑based SQL with MyBatis or Spring JDBC template to gain compile‑time validation and easier maintenance.  
2. **Batch insertion** – Modify DAO to accumulate events and execute `PreparedStatement.addBatch()` with `executeBatch()` to improve Hive write throughput.