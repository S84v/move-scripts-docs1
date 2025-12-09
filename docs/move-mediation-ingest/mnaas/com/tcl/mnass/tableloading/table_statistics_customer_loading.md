# Summary
`table_statistics_customer_loading` is a Java utility that copies aggregated customer‑level statistics from an intermediate Hive table to a target Hive table. It establishes a JDBC connection to Hive, enables non‑strict dynamic partitioning, and executes a single `INSERT … SELECT` statement that preserves the `partition_date` partition column.

# Key Components
- **class `table_statistics_customer_loading`**
  - `main(String[] args)`: orchestrates configuration loading, logging setup, Hive connection, execution of Hive session settings, and the INSERT‑SELECT operation.
  - `catchException(Exception e, Connection c, Statement s)`: logs the exception, performs cleanup, and exits with status 1.
  - `closeAll(Connection c, Statement s)`: delegates resource cleanup to `hive_jdbc_connection` utilities.

# Data Flow
| Stage | Source | Destination | Transformation |
|-------|--------|-------------|----------------|
| Input | Properties file (path supplied as `args[0]`) | – | Reads Hive host, port, DB name, source and destination table names. |
| Processing | Hive intermediate table `src_tablename` (configured via properties) | Hive target table `dest_tablename` (configured via properties) | Executes `INSERT INTO … PARTITION(partition_date) SELECT … FROM src_table`. No data mutation beyond column projection. |
| Output | – | Hive target table (partitioned by `partition_date`) | Data persisted in Hive. |
| Side Effects | Log file (path supplied as `args[1]`) | – | Writes operational logs; closes JDBC resources. |

External services:
- Hive server accessed via JDBC (`hive_jdbc_connection.getJDBCConnection`).
- Local filesystem for properties and log files.

# Integrations
- **`hive_jdbc_connection`**: utility class providing JDBC connection creation and resource cleanup.
- **Other table‑loading jobs** (e.g., `siminventory_insertion`, `table_statistics_calldate_loading`) that populate the same intermediate tables; this job consumes those results.
- **Production orchestration** (e.g., Oozie, Airflow, or custom shell scripts) that invoke the JAR with appropriate arguments.

# Operational Risks
- **Missing or malformed properties file** → job aborts; mitigated by validating required keys before use.
- **Hive connection failure** → unhandled `SQLException` leads to immediate exit; mitigated by retry logic or circuit‑breaker wrapper.
- **Dynamic partition limits**: `SET hive.exec.max.dynamic.partitions.pernode=1000` may be insufficient for large date ranges; monitor Hive logs and adjust as needed.
- **Hard‑coded SQL**: schema changes (column addition/removal) will cause runtime failures; mitigate by externalizing column list or using `SELECT *` with explicit column mapping.

# Usage
```bash
# Build (assuming Maven)
mvn clean package

# Run
java -cp target/mnaas-*.jar com.tcl.mnass.tableloading.table_statistics_customer_loading \
    /path/to/config.properties \
    /var/log/mnaas/table_statistics_customer_loading.log
```
- `config.properties` must contain keys: `HIVE_HOST`, `HIVE_JDBC_PORT`, `dbname`, `table_statistics_customer_aggr_inter_tblname`, `table_statistics_customer_aggr_tblname`.
- Log file will be appended; ensure the process user has write permission.

# Configuration
| Property | Description | Example |
|----------|-------------|---------|
| `HIVE_HOST` | Hive server hostname or IP | `hive-prod.example.com` |
| `HIVE_JDBC_PORT` | Hive Thrift JDBC port | `10000` |
| `dbname` | Hive database containing source & destination tables | `mnaas` |
| `table_statistics_customer_aggr_inter_tblname` | Source intermediate table name | `customer_stats_intermediate` |
| `table_statistics_customer_aggr_tblname` | Destination aggregated table name | `customer_stats_aggregated` |

No environment variables are read directly; all configuration is supplied via the properties file.

# Improvements
1. **Parameter Validation** – Add a validation routine that checks presence and non‑emptiness of all required properties before establishing a connection.
2. **Retry & Back‑off Logic** – Wrap the JDBC connection and statement execution in a retry mechanism with exponential back‑off to improve resilience against transient Hive outages.