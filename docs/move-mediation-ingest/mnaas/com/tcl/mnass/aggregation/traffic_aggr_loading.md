# Summary
`traffic_aggr_loading` is a command‑line Java utility that loads aggregated traffic data from an intermediate Hive table into a final Hive traffic‑summary table. It reads a Hive JDBC configuration file, a process name (`hourly` or `daily`), and a log file path, establishes a Hive JDBC connection, sets Hive session parameters, builds and executes an INSERT‑SELECT statement with a LEFT OUTER JOIN to a country lookup table, and logs execution. All JDBC resources are closed on normal completion or error.

# Key Components
- **class `traffic_aggr_loading`**
  - `public static Logger logger` – Java logger for the class.
  - `public static FileHandler fileHandler` – Rotating file handler for log output.
  - `public static void main(String[] args)` – Entry point; parses arguments, loads properties, configures logging, opens Hive JDBC connection, sets Hive session variables, builds and executes the appropriate INSERT‑SELECT query based on `processname`.
  - `public static void catchException(Exception e, Connection c, Statement s)` – Logs exception, closes resources, exits with status 1.
  - `public static void closeAll(Connection c, Statement s)` – Delegates to `hive_jdbc_connection` utility to close statement, connection, and file handler.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1 | `args[0]` – path to properties file (JDBC host/port, DB name, table names) | `Properties.load` | In‑memory configuration |
| 2 | `args[1]` – process name (`hourly`|`daily`) | Determines source/destination tables and query template | Selected table names |
| 3 | `args[2]` – log file path | `FileHandler` attached to `logger` | Log file written |
| 4 | Hive JDBC host/port from properties | `hive_jdbc_connection.getJDBCConnection` | Open `java.sql.Connection` to Hive |
| 5 | Hive session variables (`hive.exec.dynamic.partition.mode`, `hive.exec.max.dynamic.partitions.pernode`) | Executed via `Statement.execute` | Session configured |
| 6 | Constructed INSERT‑SELECT SQL (joins intermediate table with `mnaas.country_lookup`) | Executed via `Statement.execute` | Data inserted into destination Hive table (partitioned by `partition_date`) |
| 7 | On success or failure | `closeAll` invoked in `finally` block | JDBC resources released, file handler closed |

External services:
- Hive server (via JDBC)
- Local filesystem for log file
- `mnaas.country_lookup` Hive table (lookup data)

# Integrations
- **`hive_jdbc_connection`** utility class (not shown) provides connection handling and resource‑cleanup methods.
- **Other aggregation utilities** (`TrafficSummaryAggrLoading`, `traffic_aggr_adhoc_aggregation`, `traffic_aggr_dups_remove`) share the same property file schema and may be orchestrated sequentially in a batch workflow.
- **Hive metastore** – required for dynamic partition creation and lookup table access.

# Operational Risks
- **Hard‑coded Hive session settings** – changes to Hive version may require updates; mitigate by externalizing to properties.
- **No partition list handling** – utility processes a single date range per run; missing dates cause data gaps. Ensure upstream scripts generate correct arguments.
- **SQL injection risk** – table names are concatenated from properties; restrict property values to known safe identifiers.
- **Unbounded log file growth** – `FileHandler` opened in append mode without rotation size limit; configure rotation or log rotation policy.
- **Process termination on first error** – `System.exit(1)` aborts the entire batch; consider retry or continue logic for large batch runs.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.aggregation.traffic_aggr_loading \
    /path/to/mnaas.properties \
    hourly \
    /var/log/traffic_aggr_loading.log
```
- Replace `hourly` with `daily` for daily aggregation.
- Ensure the classpath includes Hive JDBC driver and the `hive_jdbc_connection` library.

# Configuration
Properties file (`mnaas.properties`) must contain:
- `HIVE_JDBC_PORT` – Hive server port.
- `HIVE_HOST` – Hive server hostname or IP.
- `dbname` – Target Hive database name.
- `traffic_details_aggr_hourly_inter_tblname` – Source intermediate hourly table.
- `traffic_details_aggr_hourly_tblname` – Destination hourly table.
- `traffic_details_aggr_daily_inter_tblname` – Source intermediate daily table.
- `traffic_details_aggr_daily_tblname` – Destination daily table.

# Improvements
1. **Externalize Hive session parameters** to the properties file to allow environment‑specific tuning without code changes.  
2. **Implement partition list handling** (read a file of dates) and loop over partitions, similar to other aggregation utilities, to enable bulk processing in a single execution.