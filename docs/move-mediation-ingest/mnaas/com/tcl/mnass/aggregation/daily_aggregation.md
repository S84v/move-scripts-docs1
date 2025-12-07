# Summary
`daily_aggregation` is a Java command‑line utility that performs incremental daily aggregation of various telecom data domains (activations, actives, tolling, SIM inventory, and MSISDN‑level usage). It reads a Hive connection properties file, an aggregation control properties file, and runtime arguments to determine the processing type, then executes Hive/Impala INSERT‑SELECT statements with dynamic partitioning, updates the control file with the latest processed date, and writes partition values to a supplied file.

# Key Components
- **class `daily_aggregation`**
  - `public static void main(String[] args)`: entry point; loads properties, establishes Hive JDBC connection, selects processing branch, builds and executes aggregation queries, updates control file, logs progress.
  - `catchException(Exception, Connection, Statement, BufferedWriter)`: logs exception, closes resources, exits with error.
  - `closeAll(Connection, Statement, BufferedWriter)`: safely closes JDBC statement, connection, file handler, and writer.
- **Static members**
  - `Logger logger`, `FileHandler fileHandler`, `Formatter simpleFormatter`: logging infrastructure.
  - Configuration strings (`HiveHost`, `HiveJdbcPort`, `dbName`, table names, query templates, etc.).
- **External utility**
  - `com.tcl.hive.jdbc.hive_jdbc_connection`: provides `getJDBCConnection`, `closeStatement`, `closeConnection`, `closeFileHandler`.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1 | `args[0]` – Hive properties file<br>`args[1]` – aggregation control properties file<br>`args[2]` – process name (`activations`, `actives`, `tolling`, `siminventory`, `msisdn_level_daily_usage_aggr`)<br>`args[3]` – path for partition values file (used only for `activations`)<br>`args[4]` – log file path<br>`args[5]` – optional partitions list file (used only for `msisdn_level_daily_usage_aggr`) | Load properties, configure logger, open Hive JDBC connection. | In‑memory configuration objects. |
| 2 | Hive tables defined in properties (source & destination) | Execute `SELECT max(inserttime)` to obtain `maxTrafficDate`. For `activations` also query distinct `partition_date` values between `minTrafficDate` and `maxTrafficDate`. | Updated `maxTrafficDate`; for `activations` writes partition dates to `args[3]`. |
| 3 | Determined date range (`minTrafficDate` → `maxTrafficDate`) | Build domain‑specific `INSERT INTO … PARTITION(partition_date) SELECT … GROUP BY …` query and execute via JDBC. For `msisdn_level_daily_usage_aggr` iterate over partition list, drop existing partition, then insert aggregated rows. | Destination Hive tables populated with aggregated rows; partitions created/dropped. |
| 4 | After successful aggregation | Update `aggr_mindate` in `MNAASAggrProperties` to `maxTrafficDate` and persist back to `args[1]`. | Control file reflects new high‑water mark for next run. |
| 5 | Throughout | Logging to file handler; resource cleanup in `finally`. | Log file contains execution trace. |

# Integrations
- **Hive/Impala**: Uses `hive_jdbc_connection` to run HiveQL/ImpalaQL statements against the Hive metastore defined by `HIVE_HOST` and `HIVE_JDBC_PORT`.
- **Control Properties**: Shared with other aggregation jobs (`business_trend_aggregation`, `business_trend_loading`, etc.) to coordinate incremental processing.
- **External Scripts**: Typically invoked by a shell/cron wrapper that supplies the five required arguments and optionally a partition list file for MSISDN usage aggregation.
- **Logging Infrastructure**: Writes to a rotating log file consumed by monitoring/alerting systems.

# Operational Risks
- **Unbounded `maxTrafficDate`**: If source table contains future timestamps, the job may process incomplete data. *Mitigation*: Validate `maxTrafficDate` against current system date.
- **SQL Injection via Properties**: Table names and dates are concatenated directly into queries. *Mitigation*: Sanitize/whitelist property values or use prepared statements where possible.
- **Resource Leak on Exception**: `catchException` calls `System.exit(1)` after `closeAll`, but if `bw` is null it may throw NPE. *Mitigation*: Null‑check before closing.
- **Partition List File Missing**: For `msisdn_level_daily_usage_aggr` the code assumes `args[5]` exists; missing file causes `FileNotFoundException`. *Mitigation*: Validate argument count and file existence before use.
- **Concurrent Runs**: Multiple instances could update the same control file leading to race conditions. *Mitigation*: Use file locking or schedule jobs to avoid overlap.

# Usage
```bash
java -cp daily_aggregation.jar \
  com.tcl.mnass.aggregation.daily_aggregation \
  /path/to/hive.properties \
  /path/to/aggregation_control.properties \
  activations \
  /tmp/partition_dates.txt \
  /var/log/daily_aggregation.log
# For MSISDN usage aggregation (requires extra arg):
java -cp daily_aggregation.jar \
  com.tcl.mnass.aggregation.daily_aggregation \
  /path/to/hive.properties \
  /path/to/aggregation_control.properties \
  msisdn_level_daily_usage_aggr \
  /dev/null \
  /var/log/daily_aggregation.log \
  /path/to/partition_list.txt
```
Debugging: run with `-Djava.util.logging.config.file=logging.properties` to adjust log level; attach a debugger to `main`.

# Configuration
- **Hive properties file** (`args[0]`):  
  - `HIVE_HOST` – Hive server hostname  
  - `HIVE_JDBC_PORT` – JDBC port  
  - `dbname` – target database name  
  - Domain‑specific table names, e.g., `activations_daily_tblname`, `activations_aggr_daily_tblname`, etc.
- **Aggregation control properties file** (`args[1]`):  
  - `aggr_mindate` – last processed `inserttime` (ISO‑8601 string). Updated after each successful run.
- **Runtime arguments** (`args[2]`–`args[5]`): process selector, output file for partition dates, log file path, optional partition list file.

# Improvements
1. **Parameter Validation & Centralized Config** – Introduce a configuration class that validates required properties, checks argument count, and encapsulates table name resolution to prevent malformed SQL.
2. **Prepared Statements & Sanitization** – Refactor query construction to use `PreparedStatement` with bind variables for dates and partition