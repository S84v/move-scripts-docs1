# Summary
`com.tcl.mnass.aggregation.siminventory_aggr_insertion` is a command‑line Java utility that reads a Hive connection properties file and a partition date, then executes a Hive INSERT‑SELECT statement to aggregate SIM inventory data from a source table into a destination table with dynamic partitioning. Execution details are logged to a rotating file handler and all JDBC resources are closed on completion or error.

# Key Components
- **class `siminventory_aggr_insertion`**
  - `main(String[] args)`: orchestrates property loading, logger setup, JDBC connection, query execution, and cleanup.
  - `catchException(Exception, Connection, Statement)`: logs the exception, closes resources, and exits with status 1.
  - `closeAll(Connection, Statement)`: delegates to `hive_jdbc_connection` helpers to close statement, connection, and file handler.
- **Static fields**
  - `logger`, `fileHandler`, `simpleFormatter`: Java Util Logging components.
  - Hive configuration strings (`hiveDynamicPartitionMode`, `partitionsPernode`).
  - Connection parameters (`HIVE_HOST`, `HIVE_JDBC_PORT`, `db`).
  - Table names (`src_tableName`, `dest_tableName`).
  - `query`: the dynamically built INSERT‑SELECT statement.

# Data Flow
| Stage | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| 1 | Properties file (path supplied as `args[0]`) | Load Hive host, port, DB name, source & destination table names. | In‑memory configuration values. |
| 2 | Partition date (`args[1]`) | Used in WHERE clause of the INSERT‑SELECT. | Partition filter string. |
| 3 | Log file path (`args[2]`) | Initialise `FileHandler` for logging. | Log file on local filesystem. |
| 4 | JDBC connection (via `hive_jdbc_connection.getJDBCConnection`) | Connect to Hive/Impala server. | Live `Connection` object. |
| 5 | SQL execution (`stmt.execute`) | Set Hive dynamic partition mode, set max partitions per node, run INSERT‑SELECT. | Data written to `dest_tableName` partition `partition_date`. |
| 6 | Cleanup | Close `Statement`, `Connection`, and `FileHandler`. | Release resources, flush logs. |

External services:
- Hive/Impala server (JDBC endpoint).
- Local filesystem for logging.

# Integrations
- **`hive_jdbc_connection`**: utility class providing JDBC connection creation and resource‑close helpers.
- **Other aggregation utilities** (`daily_aggregation`, `high_spend_aggregation`, etc.) share the same properties file format and logging conventions, enabling coordinated batch runs via external orchestration (e.g., Oozie, Airflow, cron).
- Destination table is consumed downstream by reporting or analytics pipelines that read the aggregated SIM inventory partitions.

# Operational Risks
- **Hard‑coded SQL syntax errors** (missing column alias, extra comma) can cause job failure; mitigate by unit‑testing query generation.
- **No validation of input arguments**; malformed paths or partition dates cause `FileNotFoundException` or empty result sets. Add argument validation.
- **Static logger configuration may cause file descriptor leaks** if multiple instances run concurrently; ensure unique log file names or rotate by size.
- **Dynamic partition mode set to `nonstrict` without explicit `SET hive.exec.max.dynamic.partitions`** may exceed cluster limits; monitor Hive server metrics.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.aggregation.siminventory_aggr_insertion \
    /path/to/mnaas.properties \
    2024-11-30 \
    /var/log/mnaas/siminventory_aggr.log
```
- `<classpath>` must include compiled classes and `hive_jdbc_connection` JARs.
- Verify that the Hive tables exist and the partition date is present in the source table.

# Configuration
- **Properties file** (first argument) must contain:
  - `HIVE_HOST` – Hive server hostname.
  - `HIVE_JDBC_PORT` – Hive JDBC port.
  - `dbname` – Target Hive database.
  - `Move_siminventory_status_tblname` – Source table.
  - `Move_siminventory_status_aggr_tblname` – Destination table.
- No environment variables are referenced directly; all configuration is property‑driven.

# Improvements
1. **Parameter validation & usage help** – implement a method to check argument count, file existence, and partition date format; print usage instructions on failure.
2. **SQL construction safety** – use prepared statements or a query builder to avoid injection and to handle escaping of partition values; also correct the malformed SELECT list (remove trailing comma, add missing column aliases).