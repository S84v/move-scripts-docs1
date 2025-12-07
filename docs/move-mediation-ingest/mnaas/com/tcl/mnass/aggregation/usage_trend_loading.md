# Summary
`usage_trend_loading` is a command‑line Java utility that copies aggregated daily usage‑trend data from an intermediate Hive table (`usage_trend_inter_daily_aggr_tblname`) to a final Hive table (`usage_trend_daily_aggr_tblname`). It reads a Hive JDBC configuration file and a log file path, establishes a Hive JDBC connection, sets Hive dynamic‑partition session parameters, executes a single `INSERT … SELECT` statement, and logs execution. All JDBC resources are closed on completion or error.

# Key Components
- **class `usage_trend_loading`**
  - `main(String[] args)`: orchestrates configuration loading, logger setup, JDBC connection, session configuration, and execution of the INSERT statement.
  - `catchException(Exception, Connection, Statement)`: logs the exception, closes resources, and exits with status 1.
  - `closeAll(Connection, Statement)`: delegates resource cleanup to `hive_jdbc_connection` helper methods.
- **Static fields**
  - `logger`, `fileHandler`, `simpleFormatter`: Java Util Logging components.
  - Hive session strings: `hiveDynamicPartitionMode`, `partitionsPernode`.
  - Connection parameters: `HIVE_HOST`, `HIVE_JDBC_PORT`, `db`, `src_tablename`, `dest_tablename`.
  - `insert`: final INSERT‑SELECT SQL string.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1 | Properties file (path supplied as `args[0]`) | Load Hive connection parameters and table names. | In‑memory configuration variables. |
| 2 | Log file path ( `args[1]` ) | Initialise `FileHandler` and attach to `logger`. | Log file written with execution details and errors. |
| 3 | Hive JDBC connection (host/port from properties) | `hive_jdbc_connection.getJDBCConnection`. | Open JDBC `Connection`. |
| 4 | Session settings | Execute `SET hive.exec.dynamic.partition.mode=nonstrict` and `SET hive.exec.max.dynamic.partitions.pernode=1000`. | Hive session configured for dynamic partitions. |
| 5 | INSERT‑SELECT statement | `stmt.execute(insert)`. | Data copied from `src_tablename` to `dest_tablename` with partitioning on `partition_date`. |
| 6 | Cleanup | `closeAll` closes `Statement`, `Connection`, and `FileHandler`. | Resources released. |

External services: Hive server accessed via JDBC.

# Integrations
- **`hive_jdbc_connection`**: utility class providing `getJDBCConnection` and resource‑cleanup methods.
- **Hive tables**: reads from `usage_trend_inter_daily_aggr_tblname` (intermediate aggregation) and writes to `usage_trend_daily_aggr_tblname` (final daily aggregation) within the database defined by `dbname`.
- **Logging infrastructure**: writes to a file specified at runtime; can be consumed by monitoring/alerting tools.

# Operational Risks
- **Hard‑coded SQL**: No column list validation; schema changes can cause runtime failures. *Mitigation*: externalize column list or use schema‑evolution checks.
- **Single‑statement execution**: Failure of the INSERT aborts the entire batch without partial rollback. *Mitigation*: wrap in Hive transaction (if supported) or implement idempotent staging.
- **Unvalidated arguments**: Missing or malformed args cause `ArrayIndexOutOfBoundsException`. *Mitigation*: add argument count validation and usage help.
- **Resource leakage on unexpected exceptions**: `catchException` calls `System.exit(1)` after closing resources, but may skip finally block for other threads. *Mitigation*: ensure all resources are closed in a `finally` block regardless of exit path.

# Usage
```bash
java -cp usage_trend_loading.jar com.tcl.mnass.aggregation.usage_trend_loading \
    /path/to/hive_properties.conf \
    /var/log/usage_trend_loading.log
```
- `args[0]`: path to a Java properties file containing `HIVE_HOST`, `HIVE_JDBC_PORT`, `dbname`, `usage_trend_inter_daily_aggr_tblname`, `usage_trend_daily_aggr_tblname`.
- `args[1]`: absolute path to the log file (will be appended).

# Configuration
Properties file keys (example):
```
HIVE_HOST= hive-prod.example.com
HIVE_JDBC_PORT=10000
dbname=telecom_dw
usage_trend_inter_daily_aggr_tblname=usage_trend_inter_daily
usage_trend_daily_aggr_tblname=usage_trend_daily
```
No environment variables are referenced directly; all configuration is read from the supplied properties file.

# Improvements
1. **Argument validation & help message** – add a method to verify argument count and display usage instructions.
2. **SQL externalization & schema safety** – move the INSERT‑SELECT statement to a template file and validate column existence at startup to protect against schema drift.