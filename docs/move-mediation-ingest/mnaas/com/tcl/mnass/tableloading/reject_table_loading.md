# Summary
`reject_table_loading` is a command‑line Java utility that loads rejected telecom records from a staging Hive table into a target Hive table. It selects ETL logic based on the `processname` argument (e.g., `traffic_details`, `IPVProbe_feed`, `esimhub`), builds Hive INSERT … SELECT statements (including special handling for HOL/SNG partitions), executes them via a JDBC connection, and logs activity. It is used in production to persist rejected data for downstream analysis and reporting.

# Key Components
- **class `reject_table_loading`** – entry point; parses CLI arguments, configures logging, obtains Hive JDBC connection, and drives processing.
- **`main(String[] args)`** – orchestrates argument handling, connection setup, dynamic partition settings, process‑specific query generation, execution, and exception handling.
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, exits with error code.
- **`closeAll(Connection, Statement)`** – delegates resource cleanup to `hive_jdbc_connection` utilities.
- **Static configuration strings** – Hive dynamic partition mode/per‑node settings.

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | CLI arguments: `processname`, `dbName`, `src_tableName`, `dest_tableName`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `logFilePath`. |
| **Processing** | 1. Open `FileHandler` for logging.<br>2. Obtain Hive JDBC `Connection` via `hive_jdbc_connection.getJDBCConnection`.<br>3. Execute `SET` statements for dynamic partitioning.<br>4. Build a process‑specific Hive `INSERT … SELECT` query (including filters for rejected records).<br>5. Execute the query via `Statement.execute`. |
| **Output** | Rows inserted into `dbName.dest_tableName` partitioned by `partition_date` (or `partition_month`). |
| **Side Effects** | - Hive tables are modified (data load).<br>- Log file is written.<br>- JDBC resources are allocated and released. |
| **External Services** | Hive server (via JDBC). |
| **Databases** | Hive metastore/database identified by `dbName`. |

# Integrations
- **`hive_jdbc_connection`** – utility class providing JDBC connection management and cleanup; shared across ingestion utilities (`raw_table_loading`, etc.).
- **Other ingestion scripts** – `reject_table_loading` is invoked after primary loads (e.g., `raw_table_loading`) to capture records that failed validation or belong to special sources (HOL/SNG, IPVProbe, esimhub).
- **Logging infrastructure** – writes to a file path supplied at runtime; can be consumed by log aggregation tools (e.g., Splunk, ELK).

# Operational Risks
- **Hard‑coded filter logic** (`customernumber='' or customernumber='NewCustomer12345' or calldate is NULL`) may miss new reject patterns → periodic review of filter criteria.
- **SQL injection risk** if arguments are not validated (table names, DB name). Mitigation: enforce whitelist or use prepared statements where possible.
- **Resource leakage** if `catchException` fails before `closeAll` → ensure `closeAll` is idempotent and robust.
- **Partition overflow**: dynamic partition mode set to `nonstrict` with high per‑node limit; could generate excessive partitions → monitor partition count and adjust limits.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.tableloading.reject_table_loading \
    traffic_details \
    mnaas \
    traffic_details_raw \
    traffic_details_reject \
    hive-host.example.com \
    10000 \
    /var/log/mnaas/reject_table_loading.log
```
- Replace placeholders with actual Hive host, port, and log file path.
- For debugging, set logger level to `FINE` or add `-Djava.util.logging.config.file=...` to control verbosity.

# Configuration
- **CLI arguments** (see Usage) provide all runtime configuration.
- **Hive connection** – host and port supplied via arguments; authentication handled inside `hive_jdbc_connection`.
- **Logging** – `FileHandler` writes to the path given as the 7th argument; uses `SimpleFormatter`.

# Improvements
1. **Parameter validation & sanitization** – implement whitelist checks for `dbName`, `src_tableName`, `dest_tableName` and reject or escape unsafe values.
2. **Externalize query templates** – move large INSERT statements to resource files (e.g., YAML or .sql) and load them at runtime to improve readability and maintainability.