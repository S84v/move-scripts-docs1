# Summary
`business_trend_loading` loads daily aggregated business‑trend metrics from an intermediate Hive table (`business_trend_inter_daily_aggr_tblname`) into a final Hive table (`business_trend_daily_aggr_tblname`) using a single INSERT … SELECT statement with dynamic partitioning. It is invoked from a shell/automation job, reads a properties file for Hive connection details and table names, logs execution, and ensures resources are closed on success or failure.

# Key Components
- **class `business_trend_loading`**
  - `public static void main(String[] args)`: entry point; parses arguments, loads properties, configures logger, establishes Hive JDBC connection, executes Hive session settings, builds and runs the INSERT statement, handles exceptions, and triggers cleanup.
  - `public static void catchException(Exception e, Connection c, Statement s)`: logs the exception, closes resources, exits with status 1.
  - `public static void closeAll(Connection c, Statement s)`: delegates to `hive_jdbc_connection` utility to close `Statement`, `Connection`, and `FileHandler`.

# Data Flow
| Stage | Source | Destination | Transformation |
|-------|--------|-------------|----------------|
| 1 | Properties file (`args[0]`) | In‑memory `Properties` object | Reads Hive host, port, DB name, source & destination table names |
| 2 | Hive JDBC connection (host/port) | `Connection` object | Established via `hive_jdbc_connection.getJDBCConnection` |
| 3 | Hive session | Executes `SET hive.exec.dynamic.partition.mode=nonstrict` and `SET hive.exec.max.dynamic.partitions.pernode=1000` | Enables non‑strict dynamic partitioning |
| 4 | Source Hive table (`src_tablename`) | Destination Hive table (`dest_tablename`) | `INSERT INTO TABLE db.dest_tablename PARTITION(partition_date) SELECT … FROM db.src_tablename` (field‑by‑field copy) |
| 5 | Log file (`args[1]`) | Append log entries | `java.util.logging` writes INFO and error messages |
| 6 | Cleanup | Closes JDBC `Statement`, `Connection`, and `FileHandler` | Ensures no resource leaks |

**Side Effects**
- Writes rows into the destination Hive table (partitioned by `partition_date`).
- Generates/updates a log file.
- May affect downstream processes that consume the destination table.

# Integrations
- **Hive JDBC driver** (`com.tcl.hive.jdbc.hive_jdbc_connection`): provides connection handling and resource‑cleanup utilities.
- **Logging subsystem**: integrates with Java `java.util.logging` to produce a persistent log file.
- **External orchestration**: typically called from a shell script or workflow manager (e.g., Oozie, Airflow) that supplies the properties file path and log file path as arguments.
- **Downstream analytics**: the populated destination table is consumed by reporting/aggregation jobs such as `business_trend_aggregation`.

# Operational Risks
- **Missing/invalid properties file** → `FileNotFoundException` → job aborts. *Mitigation*: validate arguments before execution; provide default fallback or pre‑flight check.
- **Hive connection failure** (network, authentication) → `SQLException`. *Mitigation*: implement retry logic; monitor Hive service health.
- **Schema drift** (source/target column mismatch) → INSERT fails. *Mitigation*: enforce schema versioning; add validation step before INSERT.
- **Unbounded partition growth** → excessive partitions may degrade Hive performance. *Mitigation*: enforce retention policy; monitor partition count.
- **Log file permission issues** → `FileHandler` cannot write. *Mitigation*: ensure the process user has write access to the log directory.

# Usage
```bash
# args[0] = absolute path to properties file
# args[1] = absolute path to log file (will be appended)
java -cp /path/to/jars/* com.tcl.mnass.aggregation.business_trend_loading \
     /opt/conf/business_trend_loading.properties \
     /var/log/mnaas/business_trend_loading.log
```
*Debug*: set logger level to `FINE` in the properties file or modify `logger.setLevel(Level.ALL)` to `Level.FINE`.

# Configuration
- **Properties file** (path supplied as first CLI argument) must contain:
  - `HIVE_HOST` – Hive server hostname or IP.
  - `HIVE_JDBC_PORT` – Hive Thrift port (default 10000).
  - `dbname` – Target Hive database name.
  - `business_trend_inter_daily_aggr_tblname` – Source table.
  - `business_trend_daily_aggr_tblname` – Destination table.
- No environment variables are read directly; all configuration is via the properties file.

# Improvements
1. **Add schema validation**: query `DESCRIBE` on source and destination tables, compare column lists, and abort with a clear error if mismatched.
2. **Implement retry/back‑off for Hive connection**: wrap `getJDBCConnection` in a loop with exponential back‑off to improve resilience against transient network issues.