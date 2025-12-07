# Summary
`RawTableLoading` is a command‑line Java utility that loads raw data into a Hive dimension table. It reads a process‑specific Hive DDL/DML statement from a query definition file, establishes a Hive JDBC connection, configures dynamic partition settings, optionally truncates a target table, executes the main load query, and for the *kyc_feed_load* process runs an additional snapshot query. Execution details and any errors are written to a user‑specified log file.

# Key Components
- **`RawTableLoading` (class)**
  - `logger` – Java `Logger` instance for runtime logging.
  - `fileHandler` – `FileHandler` attached to the logger for log‑file output.
  - `simpleFormatter` – `SimpleFormatter` for log formatting.
  - Static configuration strings: `HIVE_JDBC_PORT`, `HIVE_HOST`, `sqlPath`, `hiveDynamicPartitionMode`, `hiveDynamicPartitionPerNode`.
  - **`main(String[] args)`** – entry point; parses CLI arguments, configures logging, reads the query, opens a Hive JDBC connection, executes Hive statements, and handles process‑specific logic.
- **`QueryReader` (external class)**
  - Constructed with `(sqlPath, processName)`.
  - Method `queryCreator()` returns the Hive query string for the supplied process.
- **`hive_jdbc_connection` (external utility)**
  - Static method `getJDBCConnection(host, port, logger, fileHandler)` returns a `java.sql.Connection` to Hive.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | CLI arguments: `processname`, `sqlPath`, `dbName`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `logFilePath` | Assign to local variables; configure logger with `logFilePath`. | Log file created/updated. |
| 2 | `sqlPath` + `processname` | `QueryReader` parses the query definition file (key/value pairs delimited by `:==>`). | `query` string containing Hive DDL/DML. |
| 3 | `HIVE_HOST`, `HIVE_JDBC_PORT` | `hive_jdbc_connection.getJDBCConnection` opens a JDBC session to Hive. | `Connection con`. |
| 4 | `con` | Create `Statement stmt`; execute dynamic partition settings. | Hive session configured. |
| 5 | `processname` | If `kyc_feed_load`, execute `TRUNCATE TABLE mnaas.kyc_iccid_wise_country_mapping`. | Target table cleared. |
| 6 | `query` | `stmt.execute(query)` runs the main load statement. | Target Hive table populated. |
| 7 | `processname` = `kyc_feed_load` (optional) | Load snapshot query via a second `QueryReader` (`kyc_feed_snapshot`) and execute it. | Snapshot table populated. |
| 8 | End of `main` | No explicit cleanup; JVM termination. | Resources may remain open until GC. |

# Integrations
- **`QueryReader`** – supplies process‑specific Hive statements from a YAML/flat‑file query definition (`query.yml` or similar).
- **`hive_jdbc_connection`** – abstracts Hive JDBC driver loading and connection handling.
- **Hive Metastore** – accessed via JDBC to execute DDL/DML; tables such as `mnaas.kyc_iccid_wise_country_mapping` are directly modified.
- **Logging subsystem** – writes operational logs to the file path supplied on the command line; can be consumed by monitoring/alerting tools.
- **Other table‑loading utilities** (`LatLongTableLoading`, `dim_date_time_3months`) – share the same pattern (CLI args, QueryReader, Hive JDBC) and may be orchestrated together in batch workflows.

# Operational Risks
- **Missing/invalid arguments** – No validation; `ArrayIndexOutOfBoundsException` will terminate the job abruptly. *Mitigation*: add argument count checks and user‑friendly error messages.
- **Uncaught SQL exceptions** – `SQLException` is printed to `stderr` but does not affect exit status; downstream processes may assume success. *Mitigation*: propagate error, set non‑zero exit code, and log stack trace.
- **Resource leaks** – `Connection` and `Statement` are never closed; can exhaust Hive server connections under high load. *Mitigation*: use try‑with‑resources or explicit `close()` in a finally block.
- **Hard‑coded Hive partition settings** – May be unsuitable for different cluster configurations. *Mitigation*: externalize these settings to a config file or environment variables.
- **Security of credentials** – Host and port are passed as plain CLI arguments; any credentials required by `hive_jdbc_connection` are not shown but could be exposed. *Mitigation*: use secure credential stores (e.g., Kerberos, Hadoop credential provider) and avoid passing secrets on the command line.

# Usage
```bash
# Syntax
java -cp <classpath> com.tcl.mnass.tableloading.RawTableLoading \
    <processName> <queryFilePath> <dbName> <hiveHost> <hivePort> <logFilePath>

# Example
java -cp lib/*:target/raw-table-loading.jar \
    com.tcl.mnass.tableloading.RawTableLoading \
    kyc_feed_load /opt/queries/query.yml mnaas 10.133.43.97 10000 /var/log/raw_table_loading.log
```
- Replace `<classpath>` with the required JARs (Hive JDBC driver, utility classes).
- Ensure `query.yml` contains a line