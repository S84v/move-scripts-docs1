# Summary
`traffic_raw_table_without_dups_sms_only_inter_loading` is a Java utility that populates an intermediate Hive table (`traffic_details_inter_raw_daily_with_no_dups_sms_only`) with SMS‑only traffic records extracted from a raw deduplicated Hive table (`traffic_details_raw_daily_with_no_dups`). For each partition date listed in an input file, it truncates the intermediate table, then executes an `INSERT … SELECT` statement that filters rows where `sms_usage > 0`. The program runs via a Hive JDBC connection, logs all actions to a configurable log file, and exits with status 1 on any exception.

# Key Components
- **Class `traffic_raw_table_without_dups_sms_only_inter_loading`** – entry point; contains `main`, exception handling, and resource cleanup.
- **`main(String[] args)`** – parses arguments, loads properties, reads partition dates, establishes Hive JDBC connection, truncates target table, iterates over dates to run INSERT statements.
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, terminates process with exit code 1.
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` utility to close statement, connection, and file handler.
- **External utility `com.tcl.hive.jdbc.hive_jdbc_connection`** – provides static methods for obtaining JDBC connections and closing resources.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|-----------|----------------------|
| 1 | Command‑line arguments: `<properties_file> <partition_dates_file> <log_file>` | Load Java `Properties` from file. | In‑memory configuration map. |
| 2 | Partition dates file (plain text, one date per line) | Read each line into `List<String> partitionDates`. | List of partition dates. |
| 3 | Hive JDBC connection parameters (host/port) from properties | `hive_jdbc_connection.getJDBCConnection`. | Open `java.sql.Connection` to Hive. |
| 4 | `truncate_query` built from `dbname` and `traffic_details_inter_raw_daily_with_no_dups_sms_only_tblname`. | Execute `stmt.execute(truncate_query)`. | Intermediate table emptied. |
| 5 | For each `event_date` in `partitionDates` | Build `INSERT … SELECT` query filtering `sms_usage > 0` and matching `partition_date`. | Rows inserted into intermediate table. |
| 6 | Logging via `java.util.logging.Logger` to the supplied log file. | `logger.info` for each query, `logger.log` for exceptions. | Persistent audit trail. |
| 7 | Finally block | Calls `closeAll` to release JDBC resources and file handler. | Clean shutdown. |

External services:
- Hive server (via JDBC) – executes DDL/DML.
- File system – reads properties, partition list, writes log.

# Integrations
- **Pre‑processing**: Relies on `traffic_raw_table_without_dups_loading` (or similar) that creates/populates `traffic_details_raw_daily_with_no_dups` (source table).
- **Downstream**: The intermediate table populated here (`*_sms_only`) is later consumed by production loading scripts (e.g., `traffic_raw_table_without_dups_sms_only_loading`) that move data to final production tables.
- **Shared configuration**: Uses the same properties file format as other table‑loading utilities in the `move-mediation-ingest` package.

# Operational Risks
- **Unbounded partition list** – Very large date file may cause out‑of‑memory `ArrayList`. *Mitigation*: stream dates and process one at a time.
- **Truncate before insert** – If the job fails after truncation, intermediate table remains empty, causing downstream jobs to load incomplete data. *Mitigation*: perform insert into a staging table and swap after successful load.
- **Hard‑coded SQL strings** – No parameter binding; SQL injection risk if partition file is compromised. *Mitigation*: validate date format (`YYYY-MM-DD`) before use.
- **Single JDBC connection** – Long‑running connection may time out. *Mitigation*: enable keep‑alive or reconnect on failure.
- **Exit on first exception** – May leave partial data. *Mitigation*: implement retry logic and/or checkpointing.

# Usage
```bash
# Compile (if not using pre‑built jar)
javac -cp <hive-jdbc-jar>:<log4j-jar> com/tcl/mnass/tableloading/traffic_raw_table_without_dups_sms_only_inter_loading.java

# Run
java -cp .:<hive-jdbc-jar> com.tcl.mnass.tableloading.traffic_raw_table_without_dups_sms_only_inter_loading \
    /path/to/config.properties \
    /path/to/partition_dates.txt \
    /var/log/mnaas/traffic_raw_sms_only_inter.log
```
- `config.properties` must contain keys: `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`, `traffic_details_inter_raw_daily_with_no_dups_sms_only_tblname`, `traffic_details_raw_daily_with_no_dups_sms_only_tblname`, `traffic_details_raw_daily_with_no_dups_tblname`.
- `partition_dates.txt` – one Hive partition date per line (format compatible with Hive `partition_date` column).

# Configuration
| Property | Description |
|----------|-------------|
| `dbname` | Hive database name. |
| `HIVE_JDBC_PORT` | Port for HiveServer2 JDBC. |
| `HIVE_HOST` | Hostname of HiveServer2. |
| `IMPALAD_JDBC_PORT` | (unused in this class, present for consistency). |
| `IMPALAD_HOST` | (unused). |
| `traffic_details_inter_raw_daily_with_no_dups_sms_only_tblname` | Target intermediate table. |
| `traffic_details_raw_daily_with_no_dups_sms_only_tblname` | (defined but not used; retained for parity). |
| `traffic_details_raw_daily_with_no_dups_tblname` | Source raw deduplicated table. |

# Improvements
1. **Streamlined Partition Processing** – Replace the `List<String>` accumulation with a line‑by‑line loop that truncates once, then processes each date, reducing memory footprint.
2. **Transactional Load** – Use a temporary staging table and `INSERT OVERWRITE` or `ALTER TABLE ... RENAME` after successful completion to avoid data loss on failure.