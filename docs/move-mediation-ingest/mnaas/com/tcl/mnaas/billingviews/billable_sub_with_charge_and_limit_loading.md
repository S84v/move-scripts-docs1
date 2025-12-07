# Summary
Java utility `billable_sub_with_charge_and_limit_loading` loads billing view data from a temporary Hive table into the production Hive table for a specified partition date. It drops existing partitions (including a fallback for early‑month dates) and inserts refreshed rows, logging all actions.

# Key Components
- **billable_sub_with_charge_and_limit_loading (class)**
  - `main(String[] args)`: orchestrates property loading, Hive connection, partition management, and data insertion.
  - `catchException(Exception, Connection, Statement)`: logs error, cleans resources, exits with status 1.
  - `closeAll(Connection, Statement)`: delegates cleanup to `hive_jdbc_connection`.
- **Logger & FileHandler**
  - Configured to write detailed logs to a user‑provided file.
- **Hive JDBC utilities (`hive_jdbc_connection`)**
  - Provides `getJDBCConnection`, `closeStatement`, `closeConnection`, `closeFileHandler`.

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| 1 | `args[0]` – path to properties file | Load DB name, Hive/Impala host/port, table names | In‑memory configuration |
| 2 | `args[1]` – partition identifier (string) | Used in DROP and INSERT statements | Hive partition name |
| 3 | `args[2]` – log file path | Initialise `FileHandler` | Log file on local FS |
| 4 | `args[3]` – current day (DD) | Determines early‑month partition cleanup | Conditional DROP |
| 5 | `args[4]` – previous month date (YYYY‑MM‑DD) | Used for early‑month DROP | Conditional DROP |
| 6 | Hive JDBC connection | Execute Hive statements: set dynamic partition mode, drop partitions, insert data | Updated Hive tables, new partition data |
| 7 | `billable_sub_with_charge_and_limit_temp_tblname` | Source temporary table for INSERT | Rows copied to production table |

# Integrations
- **Hive Metastore** – accessed via JDBC for DDL/DML.
- **Impala (commented out)** – alternative execution path.
- **`hive_jdbc_connection` library** – encapsulates connection handling.
- **External configuration** – `mnaas_prod.properties` supplies default values; runtime overrides via command‑line properties file.
- **Logging subsystem** – writes to a file consumed by operational monitoring.

# Operational Risks
- **Hard‑coded SQL strings** – susceptible to injection if partition values are malformed. *Mitigation*: validate partition inputs against a whitelist or regex.
- **No transaction control** – partial failures leave tables in inconsistent state. *Mitigation*: wrap DROP/INSERT in a Hive transaction (if supported) or implement idempotent logic.
- **Static logger level `ALL`** – may generate excessive log volume. *Mitigation*: configure level via property.
- **Immediate `System.exit(1)` on any exception** – aborts downstream jobs. *Mitigation*: propagate error codes or retry logic.
- **Resource leakage on unexpected termination** – relies on finally block; abrupt kill may leave file handles open. *Mitigation*: use try‑with‑resources where possible.

# Usage
```bash
java -cp /path/to/mnaas.jar \
     com.tcl.mnaas.billingviews.billable_sub_with_charge_and_limit_loading \
     /opt/mnaas/conf/billable_sub.properties \
     2024-09-01 \
     /var/log/mnaas/billable_sub.log \
     01 \
     2024-08-31
```
- `billable_sub.properties` must contain keys: `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`, `billable_sub_with_charge_and_limit_tblname`, `billable_sub_with_charge_and_limit_temp_tblname`.

# Configuration
- **Properties file (first argument)**
  - `dbname` – Hive database name.
  - `HIVE_JDBC_PORT`, `HIVE_HOST` – Hive server connection.
  - `IMPALAD_JDBC_PORT`, `IMPALAD_HOST` – (optional) Impala connection.
  - `billable_sub_with_charge_and_limit_tblname` – Target production table.
  - `billable_sub_with_charge_and_limit_temp_tblname` – Source temporary table.
- **Environment**
  - Java 8+ runtime.
  - Hive client JARs on classpath (as defined in `mnaas_prod.properties`).
  - Network access to Hive/Impala hosts.

# Improvements
1. **Parameter Validation & Sanitization** – implement strict regex checks on partition strings and day values to prevent malformed SQL.
2. **Refactor to Use Prepared Statements / Hive Transaction** – replace concatenated SQL with parameterised statements and, if supported, wrap DDL/DML in a transaction to guarantee atomicity.