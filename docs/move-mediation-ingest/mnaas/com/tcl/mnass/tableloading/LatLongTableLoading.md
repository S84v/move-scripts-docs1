# Summary
`LatLongTableLoading` is a command‑line Java utility that loads a Hive “lat_long” dimension table. It establishes a Hive JDBC connection, sets dynamic partition parameters, reads a Hive DDL/DML statement from a YAML query file (`query.yml` under the key `lat_long_table_loading`), executes the statement, and logs all actions to a user‑specified log file. The process exits with status 0 on success or 1 on failure.

# Key Components
- **`LatLongTableLoading` (class)**
  - `main(String[] args)`: orchestrates argument parsing, logger setup, Hive connection, query execution, and cleanup.
  - `catchException(Exception, Connection, Statement)`: logs the exception, performs cleanup, and terminates the JVM with exit code 1.
  - `closeAll(Connection, Statement)`: delegates resource closure to `hive_jdbc_connection` utilities.
- **`logger` (static Logger)**: writes operational messages to the supplied log file.
- **`fileHandler` / `simpleFormatter`**: configure file‑based logging.
- **`hiveDynamicPartitionMode` / `hiveDynamicPartitionPerNode` (static strings)**: Hive session settings for non‑strict dynamic partitioning.
- **`QueryReader` (external class)**
  - Instantiated with the path to `query.yml` and the query key `lat_long_table_loading`.
  - `queryCreator()`: returns the Hive SQL string to be executed.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1 | Command‑line args: `<hive_host> <hive_port> <log_file_path>` | Initialize logger, open `FileHandler`. | Log file created/appended. |
| 2 | `query.yml` (YAML file) | `QueryReader` extracts the SQL statement for key `lat_long_table_loading`. | SQL string (`query`). |
| 3 | Hive JDBC connection (host/port) | Execute `SET hive.exec.dynamic.partition.mode=nonstrict` and `SET hive.exec.max.dynamic.partitions.pernode=1000`. | Session configuration applied. |
| 4 | Hive SQL (`query`) | `Statement.execute(query)`. | Populates/truncates the target lat/long Hive table. |
| 5 | Exceptions (any) | `catchException` logs, closes resources, exits with code 1. | Process termination on error. |
| 6 | Normal completion | `finally` block invokes `closeAll`. | All JDBC resources and file handler closed; process exits with code 0. |

External services:
- Hive server (via JDBC).
- File system (log file, YAML query file).

# Integrations
- **`hive_jdbc_connection`**: Provides static methods for obtaining and closing Hive JDBC connections and associated resources.
- **`QueryReader`**: Shared utility for loading parameterized Hive queries from a central YAML repository used by multiple table‑loading utilities.
- **Other table‑loading utilities** (`cdr_buid_search`, `cdr_buid_search_temp`, `dim_date_time_3months`) follow the same pattern, enabling orchestration via shell scripts or workflow engines (e.g., Oozie, Airflow).

# Operational Risks
- **Hard‑coded query file path** (`/app/hadoop_users/MNAAS/MNAAS_Property_Files/query.yml`).  
  *Mitigation*: Externalize path via environment variable or command‑line argument.
- **No validation of command‑line arguments** (e.g., missing or malformed host/port).  
  *Mitigation*: Add argument count check and format validation; exit with clear error code.
- **Uncaught `NullPointerException` if `QueryReader` returns `null`.**  
  *Mitigation*: Verify query non‑null before execution; log explicit error.
- **Single point of failure on Hive connection; no retry logic.**  
  *Mitigation*: Implement configurable retry with exponential back‑off.
- **Logging at `Level.ALL` may generate excessive log volume.**  
  *Mitigation*: Use `Level.INFO` for normal operation; reserve `Level.FINE`/`Level.FINER` for debug.

# Usage
```bash
# Syntax
java -cp <classpath> com.tcl.mnass.tableloading.LatLongTableLoading <HIVE_HOST> <HIVE_JDBC_PORT> <LOG_FILE>

# Example
java -cp lib/*:target/latlong-loader.jar \
     com.tcl.mnass.tableloading.LatLongTableLoading \
     hive-prod.example.com 10000 /var/log/latlong_loader.log
```
- Ensure the Hive JDBC driver JAR is on the classpath.
- Verify that `query.yml` contains a valid `lat_long_table_loading` entry.

# Configuration
- **Command‑line arguments**: Hive host, Hive JDBC port, absolute path to log file.
- **Static Hive session settings**:  
  - `hive.exec.dynamic.partition.mode=nonstrict`  
  - `hive.exec.max.dynamic.partitions.pernode=1000`
- **Query definition file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/query.yml` (YAML) with key `lat_long_table_loading`.
- **Logging**: Uses `java.util.logging` with `SimpleFormatter`.

# Improvements
1. **Parameterize query file location** – accept the YAML path as an argument or environment variable to avoid hard‑coding.
2. **Add robust argument validation and retry logic** – ensure graceful handling of missing/invalid inputs and transient Hive connectivity issues.