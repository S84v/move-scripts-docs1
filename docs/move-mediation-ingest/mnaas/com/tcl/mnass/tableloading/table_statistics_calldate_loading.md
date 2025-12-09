# Summary
`table_statistics_calldate_loading` is a Java utility that loads aggregated call‑date statistics from an intermediate Hive table into a target Hive table. It establishes a JDBC connection to Hive, enables non‑strict dynamic partitioning, and executes a single `INSERT … SELECT` statement that copies all columns—including the `partition_date` partition column—into the destination table.

# Key Components
- **class `table_statistics_calldate_loading`** – entry point containing `main`.
- **`main(String[] args)`** – parses a properties file (arg 0) and a log file path (arg 1), configures logging, reads Hive connection parameters, builds and executes the Hive SQL.
- **`catchException(Exception, Connection, Statement)`** – logs the exception, closes resources, exits with status 1.
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` utility for safe cleanup of JDBC objects and file handler.
- **Static SQL fragments** – `hiveDynamicPartitionMode`, `partitionsPernode`, and the `INSERT` statement built from property values.

# Data Flow
| Stage | Source | Destination | Action |
|-------|--------|-------------|--------|
| Input | Properties file (path supplied as `args[0]`) | In‑memory configuration variables | Load Hive host/port, DB name, source and destination table names |
| Input | Log file path (args[1]) | `java.util.logging.FileHandler` | Append logs |
| Process | Hive source table (`db.src_tablename`) | Hive destination table (`db.dest_tablename`) | `INSERT INTO … PARTITION(partition_date) SELECT … FROM src` |
| Side‑effects | Hive metastore | Updated destination table partitions | Dynamic partition creation |
| External services | Hive server (via JDBC) | – | JDBC connection, statement execution |
| Cleanup | JDBC `Connection`, `Statement`, `FileHandler` | – | Closed in `finally` block |

# Integrations
- **`hive_jdbc_connection`** – custom wrapper providing `getJDBCConnection`, `closeStatement`, `closeConnection`, `closeFileHandler`.
- **Hive Metastore** – accessed through JDBC for DDL/DML execution.
- **Logging subsystem** – integrates with Java Util Logging; log file path supplied at runtime.
- **Other ingestion jobs** – shares naming conventions (`table_statistics_calldate_aggr_inter_tblname`, `table_statistics_calldate_aggr_tblname`) with upstream aggregation processes that populate the intermediate table.

# Operational Risks
- **Hard‑coded dynamic partition settings** – may exceed cluster limits; mitigate by aligning `hive.exec.max.dynamic.partitions.pernode` with cluster capacity.
- **No validation of property values** – missing or malformed entries cause `NullPointerException` or failed SQL; add pre‑execution validation.
- **Single‑statement load** – failure of the `INSERT` aborts entire batch; consider staging or incremental loads for large partitions.
- **Uncontrolled exit (`System.exit(1)`)** – abrupt termination may leave resources dangling in orchestrators; replace with proper exception propagation.

# Usage
```bash
# Prepare properties file (e.g., /opt/conf/table_stats.properties)
# Example content:
# HIVE_HOST=HiveServer01
# HIVE_JDBC_PORT=10000
# dbname=mnaas
# table_statistics_calldate_aggr_inter_tblname=stats_calldate_inter
# table_statistics_calldate_aggr_tblname=stats_calldate

# Run
java -cp <classpath-including-hive-jdbc-and-utils> \
     com.tcl.mnass.tableloading.table_statistics_calldate_loading \
     /opt/conf/table_stats.properties \
     /var/log/table_stats_load.log
```
Debugging: set logger level to `FINE` or add `System.out.println` statements before the `INSERT` to verify generated SQL.

# Configuration
- **Properties file (required)** – keys: `HIVE_HOST`, `HIVE_JDBC_PORT`, `dbname`, `table_statistics_calldate_aggr_inter_tblname`, `table_statistics_calldate_aggr_tblname`.
- **Environment** – Java runtime, Hive JDBC driver, network access to Hive host/port.
- **Logging** – file path supplied as second CLI argument; rotates per JVM execution.

# Improvements
1. **Parameter validation & defaults** – implement a validation routine that checks required properties and provides meaningful error messages before establishing a connection.
2. **Configurable partition settings** – externalize `hive.exec.dynamic.partition.mode` and `hive.exec.max.dynamic.partitions.pernode` to the properties file, allowing per‑environment tuning without code change.