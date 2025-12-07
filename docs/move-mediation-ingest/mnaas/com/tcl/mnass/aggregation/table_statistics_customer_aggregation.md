# Summary
`table_statistics_customer_aggregation` is a command‑line Java utility that reads a Hive/Impala connection properties file and a list of partition dates, queries a daily usage aggregation Hive table for per‑customer (tcl_secs_id, serv_abbr) metrics, aggregates the results, formats them as semicolon‑delimited records, and writes the output to a specified file. Execution is logged to a rotating file handler and all JDBC resources are closed on success or failure.

# Key Components
- **class `table_statistics_customer_aggregation`**
  - `main(String[] args)`: orchestrates property loading, file I/O, JDBC connection, per‑date query execution, result aggregation, and output writing.
  - `catchException(Exception, Connection, Statement, BufferedWriter)`: logs the exception, closes resources, and exits with status 1.
  - `closeAll(Connection, Statement, BufferedWriter)`: delegates to `hive_jdbc_connection` helpers to close JDBC objects and the logger file handler, then closes the writer.
- **Static members**
  - `logger`, `fileHandler`, `simpleFormatter`: Java Util Logging components for rotating file logs.
  - Hive/Impala connection parameters (`HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`).
  - `places` and `scale`: used for rounding numeric results to 4 decimal places.
- **SQL query strings** built per partition date for each metric (MO/MT SMS, MO/MT voice minutes, data usage, user counts).

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | `args[0]` – path to properties file | Load Hive/Impala connection settings and table names | In‑memory configuration |
| 2 | `args[1]` – path to file containing partition dates (one per line) | Read all dates into `partitionDates` list | List\<String\> |
| 3 | `args[2]` – output file path | Open `BufferedWriter` for result records | File on local/DFS |
| 4 | `args[3]` – log file path | Initialize `FileHandler` for rotating logs | Log file |
| 5 | For each `event_date` in `partitionDates` | Execute eight `SELECT … GROUP BY tcl_secs_id,serv_abbr` statements against Impala via JDBC; populate hash maps; compute derived fields (total minutes, total SMS, rounded values) | In‑memory maps & aggregated values |
| 6 | Build semicolon‑delimited line per distinct key and write to output file | `bw.write(result_buffer.toString())` | Result file populated |
| 7 | On completion or error | Close JDBC `Statement`, `Connection`, logger `FileHandler`, and `BufferedWriter` | Resources released, log flushed |

External services:
- Impala (or Hive) cluster accessed via JDBC.
- Local/DFS filesystem for input, output, and log files.

# Integrations
- **`hive_jdbc_connection`** utility class (not shown) provides:
  - `getImpalaJDBCConnection` for establishing the JDBC session.
  - `closeStatement`, `closeConnection`, `closeFileHandler` for resource cleanup.
- **Other aggregation utilities** (`high_spend_loading`, `siminventory_aggr_insertion`, `table_statistics_calldate_aggregation`) share the same properties file schema and logging pattern, indicating they are part of a larger nightly ETL pipeline orchestrated by a scheduler (e.g., Oozie, Airflow, cron).

# Operational Risks
- **Unbounded memory usage**: All hash maps for a partition date are kept in memory; large cardinality of `(tcl_secs_id, serv_abbr)` could cause OOM. *Mitigation*: Stream results per key or process in smaller batches.
- **Hard‑coded rounding precision** (`places = 4`) may not meet downstream reporting requirements. *Mitigation*: Externalize precision to properties.
- **No retry logic** on JDBC failures; a transient network glitch aborts the whole run. *Mitigation*: Implement retry with exponential back‑off.
- **File handling without existence checks** may overwrite existing output/log files unintentionally. *Mitigation*: Add safety flags or backup steps.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.aggregation.table_statistics_customer_aggregation \
    /path/to/usage_trend_aggregation.properties \
    /path/to/partition_dates.txt \
    /path/to/output.csv \
    /path/to/run.log
```
- Ensure the properties file defines `usage_trend_daily_aggr_tblname`, `dbname`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.
- `partition_dates.txt` contains one partition date (e.g., `2024-11-30`) per line.

# Configuration
- **Properties file** (referenced by `args[0]`):
  - `usage_trend_daily_aggr_tblname` – source Hive table.
  - `dbname` – Hive database name.
  - `HIVE_HOST`, `HIVE_JDBC_PORT` – optional (unused in current code).
  - `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` – Impala connection details.
- **Environment**: Java 8+, JDBC driver for Impala on the classpath, network access to Impala nodes, write permission to output and log paths.

# Improvements
1. **Refactor per‑metric query handling**: Consolidate the eight separate `SELECT` statements into a single query that returns all metrics in one row (e.g., using conditional aggregation) to reduce round‑trips and simplify map management.
2. **Introduce streaming result processing**: Replace in‑memory hash maps with a single pass over a combined result set, writing each aggregated line directly to the output writer to lower memory footprint and improve scalability.