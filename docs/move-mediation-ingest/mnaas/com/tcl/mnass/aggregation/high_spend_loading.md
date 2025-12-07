# Summary
`high_spend_loading` is a command‑line Java utility that copies daily high‑spend subscriber metrics from an intermediate Hive table (`high_spend_inter_daily_aggr_tblname`) to a final Hive table (`high_spend_daily_aggr_tblname`) using a single `INSERT … SELECT` with Hive dynamic partitioning. It reads a properties file for Hive connection parameters and table names, logs execution to a rotating file, and guarantees JDBC resources are closed on success or failure.

# Key Components
- **class `high_spend_loading`**
  - `main(String[] args)`: orchestrates property loading, logger setup, Hive JDBC connection, execution of Hive session settings, and the INSERT statement.
  - `catchException(Exception, Connection, Statement)`: logs the exception, closes resources, and exits with status 1.
  - `closeAll(Connection, Statement)`: delegates to `hive_jdbc_connection` utility to close `Statement`, `Connection`, and `FileHandler`.

- **Static fields**
  - `logger`, `fileHandler`, `simpleFormatter`: Java Util Logging components.
  - Hive session configuration strings: `hiveDynamicPartitionMode`, `partitionsPernode`.
  - Connection parameters: `HIVE_HOST`, `HIVE_JDBC_PORT`, `db`, `src_tablename`, `dest_tablename`.
  - `insert`: final Hive INSERT statement built at runtime.

- **External utility**
  - `com.tcl.hive.jdbc.hive_jdbc_connection`: provides `getJDBCConnection`, `closeStatement`, `closeConnection`, `closeFileHandler`.

# Data Flow
| Stage | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| 1 | Properties file (path supplied as `args[0]`) | Load Hive host, port, DB name, source & destination table names | In‑memory configuration variables |
| 2 | Log file path (supplied as `args[1]`) | Initialise `FileHandler` and attach to `logger` | Log file on local filesystem |
| 3 | Hive JDBC connection (host/port) | Obtain `java.sql.Connection` via `hive_jdbc_connection` | Live Hive session |
| 4 | Hive session | Execute `SET hive.exec.dynamic.partition.mode=nonstrict` and `SET hive.exec.max.dynamic.partitions.pernode=1000` | Hive session configured for dynamic partitions |
| 5 | Source Hive table (`high_spend_inter_daily_aggr_tblname`) | `INSERT INTO db.dest_tablename PARTITION(partition_date) SELECT … FROM db.src_tablename` | Rows copied into destination table, partitioned by `partition_date` |
| 6 | Exceptions | Logged, resources closed, process exits with code 1 | Failure trace in log |
| 7 | Normal completion | Resources closed in `finally` block | Clean shutdown |

External services: Hive metastore accessed via JDBC; local filesystem for logs.

# Integrations
- **Upstream**: `high_spend_aggregation` (or other ETL jobs) populates the intermediate table referenced by `src_tablename`.
- **Downstream**: Consumers of the final table (`high_spend_daily_aggr_tblname`) such as reporting dashboards, analytics jobs, or downstream Hive/Impala queries.
- **Shared library**: `hive_jdbc_connection` utility used across aggregation loaders for connection management and cleanup.

# Operational Risks
- **Hard‑coded Hive session settings**: May conflict with cluster‑wide defaults; mitigate by externalising to properties.
- **Single INSERT statement without batch control**: Large partitions could cause OOM or long‑running queries; mitigate with monitoring and partition size limits.
- **No validation of command‑line arguments**: Missing/incorrect args cause `ArrayIndexOutOfBoundsException`; add argument count check.
- **Logging file growth**: Unlimited append mode may fill disk; configure log rotation or size limits.
- **Immediate `System.exit(1)` on any exception**: May abort downstream jobs; consider returning error codes to orchestrator.

# Usage
```bash
# Compile (if not already packaged)
javac -cp <hive-jdbc-jar>:<common-jar> com/tcl/mnass/aggregation/high_spend_loading.java

# Run
java -cp .:<hive-jdbc-jar>:<common-jar> \
     com.tcl.mnass.aggregation.high_spend_loading \
     /path/to/high_spend_loading.properties \
     /var/log/high_spend_loading.log
```
- `high_spend_loading.properties` must contain keys: `HIVE_HOST`, `HIVE_JDBC_PORT`, `dbname`, `high_spend_inter_daily_aggr_tblname`, `high_spend_daily_aggr_tblname`.
- The second argument is the absolute path to the log file.

# Configuration
| Property | Description | Example |
|----------|-------------|---------|
| `HIVE_HOST` | HiveServer2 hostname or IP | `hive-prod.example.com` |
| `HIVE_JDBC_PORT` | HiveServer2 JDBC port | `10000` |
| `dbname` | Hive database containing source/destination tables | `telecom_dw` |
| `high_spend_inter_daily_aggr_tblname` | Intermediate aggregation table name | `high_spend_inter_daily_aggr` |
| `high_spend_daily_aggr_tblname` | Final daily aggregation table name | `high_spend_daily_aggr` |

No environment variables are read; all configuration is property‑driven.

# Improvements
1. **Parameter validation & usage help** – Add checks for argument count and property presence; emit a concise usage message on failure.
2. **Externalise Hive session settings** – Move `hive.exec.dynamic.partition.mode` and `hive.exec.max.dynamic.partitions.pernode` to the properties file to allow cluster‑specific tuning without code changes.