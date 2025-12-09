# Summary
`traffic_raw_table_without_dups_loading` is a Java utility that reads a list of partition dates, then for each date executes an `INSERT … SELECT` Hive statement to copy deduplicated raw traffic records from an intermediate Hive table (`traffic_details_inter_raw_daily_with_no_dups`) into a production Hive table (`traffic_details_raw_daily_with_no_dups`). It uses a Hive JDBC connection, configures non‑strict dynamic partitioning, logs all actions to a file, and exits with status 1 on any error.

# Key Components
- **class `traffic_raw_table_without_dups_loading`** – entry point; orchestrates configuration, logging, JDBC connection, and data load loop.  
- **`main(String[] args)`** – parses arguments, loads properties, reads partition file, sets Hive session parameters, builds and executes the INSERT query per partition.  
- **`catchException(Exception, Connection, Statement)`** – logs exception, cleans up resources, terminates the JVM with exit code 1.  
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` helpers to close statement, connection, and file handler.  
- **`hive_jdbc_connection` (external class)** – provides static methods for obtaining a Hive JDBC connection and for resource cleanup.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Config load | `args[0]` – path to a Java `.properties` file | `Properties.load()` | DB name, Hive host/port, table names |
| 2. Partition list | `args[1]` – path to a text file | BufferedReader reads each line | `List<String> partitionDates` |
| 3. Logging setup | `args[2]` – log file path | `FileHandler` + `SimpleFormatter` attached to `Logger` | Log file with INFO/ERROR entries |
| 4. Hive connection | Hive host/port from properties | `hive_jdbc_connection.getJDBCConnection()` | `java.sql.Connection` |
| 5. Session config | None | Execute `SET hive.exec.dynamic.partition.mode=nonstrict` and `SET hive.exec.max.dynamic.partitions.pernode=1000` | Hive session ready for dynamic partitions |
| 6. Data copy per partition | Each `partitionDate` from list | Build INSERT‑SELECT SQL string; execute via `Statement.execute()` | Rows inserted into target table partition; log of query |
| 7. Cleanup | None (finally block) | Close statement, connection, file handler | Release resources |

External services:
- Hive metastore accessed via JDBC (HiveServer2).
- File system for reading properties, partition list, and writing logs.

# Integrations
- **`hive_jdbc_connection`** – shared library used across multiple table‑loading utilities for connection management.  
- **Other loading utilities** (`table_statistics_customer_loading`, `traffic_aggr_adhoc_loading`, etc.) follow the same pattern; they may be orchestrated by external workflow engines (e.g., Oozie, Airflow) that invoke the compiled JAR with appropriate arguments.  
- **Production Hive tables** – target tables are part of the `mnaas` database; downstream analytics jobs consume the loaded data.

# Operational Risks
- **SQL injection via partition file** – partition dates are concatenated directly into the query string. *Mitigation*: validate format (`YYYY-MM-DD`) before use or use prepared statements.  
- **Unbounded memory usage** – all partition dates are loaded into a `List`. Very large files could cause OOM. *Mitigation*: stream partitions and process one at a time.  
- **Hard‑coded Hive session settings** – changes to Hive configuration may require code updates. *Mitigation*: externalize these settings to the properties file.  
- **No retry logic** – a single failed partition aborts the entire run. *Mitigation*: implement per‑partition try/catch and continue, or record failed partitions for re‑run.  
- **Silent data loss on duplicate inserts** – `INSERT INTO` without `OVERWRITE` may create duplicate rows if the job is re‑run for the same partition. *Mitigation*: use `INSERT OVERWRITE` or deduplication checks.

# Usage
```bash
# Compile (Maven/Gradle) to produce traffic_raw_table_without_dups_loading.jar
java -cp traffic_raw_table_without_dups_loading.jar \
     com.tcl.mnass.tableloading.traffic_raw_table_without_dups_loading \
     /path/to/config.properties \
     /path/to/partition_dates.txt \
     /var/log/traffic_raw_loading.log
```
- `config.properties` must contain keys: `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `traffic_details_inter_raw_daily_with_no_dups_tblname`, `traffic_details_raw_daily_with_no_dups_tblname`.  
- `partition_dates.txt` – one `YYYY-MM-DD` per line.  
- Log file will be appended; monitor for `ERROR` or stack traces.

# Configuration
| Property | Description | Example |
|----------|-------------|---------|
| `dbname` | Hive database name containing source & target tables | `mnaas` |
| `HIVE_JDBC_PORT` | Port of HiveServer2 | `10000` |
| `HIVE_HOST` | Hostname or IP of HiveServer2 | `hive-prod.example.com` |
| `traffic_details_inter_raw_daily_with_no_dups_tblname` | Intermediate deduped table | `traffic_details_inter_raw_daily_with_no_dups` |
| `traffic_details_raw_daily_with_no_dups_tblname` | Target production table | `traffic_details_raw_daily_with_no_dups` |

No environment variables are read; all configuration is supplied via the properties file.

# Improvements
1. **Parameterize SQL via PreparedStatement** – replace string concatenation with `?` placeholders for `partition_date` to eliminate injection risk and improve query planning.  
2. **Add per‑partition error handling and retry** – wrap the `statement.execute(query)` call in its own try/catch, log failures, continue processing remaining partitions, and optionally write failed dates to a separate file for later re‑processing.