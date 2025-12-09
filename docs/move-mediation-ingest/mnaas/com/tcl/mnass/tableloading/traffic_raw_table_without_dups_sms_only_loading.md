# Summary
`traffic_raw_table_without_dups_sms_only_loading` is a Java command‑line utility that copies SMS‑only, deduplicated traffic records from an intermediate Hive table (`traffic_details_inter_raw_daily_with_no_dups_sms_only`) into a production Hive table (`traffic_details_raw_daily_with_no_dups_sms_only`). For each partition date listed in a supplied file, it issues an `INSERT … SELECT` Hive statement via a JDBC connection, logs progress, and exits with status 1 on any failure.

# Key Components
- **`traffic_raw_table_without_dups_sms_only_loading` (class)**
  - `main(String[] args)`: orchestrates configuration loading, input parsing, Hive connection setup, dynamic‑partition settings, per‑date insert execution, and cleanup.
  - `catchException(Exception, Connection, Statement)`: logs the exception, performs cleanup, and terminates the JVM with exit code 1.
  - `closeAll(Connection, Statement)`: delegates to `hive_jdbc_connection` helpers to close the JDBC `Statement`, `Connection`, and `FileHandler`.

- **External helper class**
  - `com.tcl.hive.jdbc.hive_jdbc_connection`: provides static methods for obtaining JDBC connections and closing resources.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | Property file path (`args[0]`) | `Properties.load` → extracts DB name, Hive/Impala host/port, table names | In‑memory configuration map |
| 2 | Partition list file (`args[1]`) | BufferedReader reads each line → `partitionDates` list | List of partition dates (String) |
| 3 | Log file path (`args[2]`) | `FileHandler` attached to `logger` | Log entries written to file |
| 4 | Hive JDBC connection | `hive_jdbc_connection.getJDBCConnection` | Live `java.sql.Connection` to Hive |
| 5 | For each `event_date` | Construct `INSERT INTO … SELECT * FROM … WHERE partition_date='event_date'` and execute via `Statement.execute` | Rows inserted into production table, Hive logs |
| 6 | End of program | `closeAll` → closes JDBC resources and log handler | Clean shutdown |

External services:
- Hive (or Impala) server accessed via JDBC.
- File system for property file, partition list, and log file.

# Integrations
- **Configuration source**: External `.properties` file referenced by the first command‑line argument; must contain keys `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`, `traffic_details_inter_raw_daily_with_no_dups_sms_only_tblname`, `traffic_details_raw_daily_with_no_dups_sms_only_tblname`.
- **Hive tables**: Reads from `dbname.traffic_details_inter_raw_daily_with_no_dups_sms_only`; writes to `dbname.traffic_details_raw_daily_with_no_dups_sms_only`.
- **Logging**: Centralized log file defined by the third argument; integrates with any downstream log aggregation (e.g., Splunk, ELK) if configured.
- **Potential alternative**: Commented code shows ability to switch to Impala JDBC (`getImpalaJDBCConnection`) without code changes.

# Operational Risks
- **Unvalidated input**: Partition dates are taken verbatim; malformed dates cause Hive query failure → mitigated by pre‑validation or try‑catch (already exits with error).
- **Hard‑coded Hive settings**: Dynamic partition mode and max partitions per node are static; changes in Hive configuration may require code update → externalize to properties.
- **No transaction control**: Each `INSERT` runs independently; partial success may leave some partitions loaded while others failed → consider wrapping in a Hive transaction or implementing idempotent re‑run logic.
- **Resource leakage on abrupt termination**: `catchException` calls `closeAll` before `System.exit(1)`, but if JVM is killed externally resources may remain open → ensure OS‑level monitoring cleans up stale connections.
- **Scalability**: Sequential per‑date inserts may be slow for large date sets; parallel execution could improve throughput but introduces concurrency concerns.

# Usage
```bash
# Compile (if not already built)
javac -cp <hive-jdbc-jar>:<log4j-jar> com/tcl/mnass/tableloading/traffic_raw_table_without_dups_sms_only_loading.java

# Run
java -cp .:<hive-jdbc-jar> com.tcl.mnass.tableloading.traffic_raw_table_without_dups_sms_only_loading \
    /path/to/config.properties \
    /path/to/partition_dates.txt \
    /var/log/traffic_raw_sms_only_loading.log
```
- `config.properties` – contains required keys (see Configuration).
- `partition_dates.txt` – one `yyyy-MM-dd` (or Hive‑compatible) date per line.
- Log file will be appended; monitor for `INFO` and `SEVERE` entries.

# Configuration
| Property key | Description | Example |
|--------------|-------------|---------|
| `dbname` | Hive database name | `mnaas` |
| `HIVE_JDBC_PORT` | HiveServer2 port | `10000` |
| `HIVE_HOST` | HiveServer2 hostname or IP | `hive-prod.example.com` |
| `IMPALAD_JDBC_PORT` | Impala port (unused unless code switched) | `21050` |
| `IMPALAD_HOST` | Impala host (unused unless code switched) | `impala-prod.example.com` |
| `traffic_details_inter_raw_daily_with_no_dups_sms_only_tblname` | Intermediate table name | `traffic_details_inter_raw_daily_with_no_dups_sms_only` |
| `traffic_details_raw_daily_with_no_dups_sms_only_tblname` | Production table name | `traffic_details_raw_daily_with_no_dups_sms_only` |

No environment variables are read directly; all runtime parameters are supplied via the three command‑line arguments.

# Improvements
1. **Externalize Hive session settings** – Move `hive.exec.dynamic.partition.*` statements to the properties file to allow environment‑specific tuning without recompilation.
2. **Batch insert & idempotency** – Replace per‑date `INSERT … SELECT` with a single multi‑partition insert using `IN ( … )` or a staging table, and add a pre‑load check to skip already‑loaded partitions, reducing runtime and preventing duplicate loads.