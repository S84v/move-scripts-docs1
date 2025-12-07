# Summary
`traffic_aggr_dups_remove` is a command‑line Java utility that removes duplicate traffic detail records by aggregating hourly or daily partitions from a source Hive table into an intermediate aggregated Hive table. It reads a Hive JDBC configuration, a file containing partition dates, and a process mode (`hourly` or `daily`). For each partition it executes a Hive INSERT‑SELECT statement that deduplicates rows (using `ROW_NUMBER` for hourly) or rolls up daily aggregates, then writes logs to a rotating file. All JDBC resources are closed on success or failure.

# Key Components
- **class `traffic_aggr_dups_remove`**
  - `public static Logger logger` – logger instance.
  - `public static FileHandler fileHandler` – rotating log file handler.
  - `public static void main(String[] args)` – entry point; parses arguments, loads properties, configures logging, establishes Hive connection, reads partition list, executes aggregation queries, handles exceptions, and closes resources.
  - `public static void catchException(Exception e, Connection c, Statement s)` – logs exception, closes resources, exits with status 1.
  - `public static void closeAll(Connection c, Statement s)` – delegates to `hive_jdbc_connection` helper to close statement, connection, and file handler.

- **External helper `com.tcl.hive.jdbc.hive_jdbc_connection`**
  - Provides static methods `getJDBCConnection`, `closeStatement`, `closeConnection`, `closeFileHandler`.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | Command‑line args: `<partitions_file> <properties_file> <process_name> <log_file>` | Load partition dates from file; load Hive connection properties from `.properties`; configure logger. | In‑memory list of partition dates; logger writes to `<log_file>`. |
| 2 | Hive JDBC host/port, DB name, table names from properties | Establish Hive JDBC connection via `hive_jdbc_connection`. | Open `Connection` and `Statement`. |
| 3 | For each partition date | Build and execute Hive SQL: <ul><li>`hourly` – CTE `t1` with `ROW_NUMBER` to keep latest version, then aggregate to hourly intermediate table.</li><li>`daily` – Simple `INSERT … SELECT … GROUP BY` to roll up daily aggregates.</li></ul> | Data inserted into destination Hive table (`traffic_details_aggr_hourly_inter_tblname` or `traffic_details_aggr_daily_inter_tblname`). |
| 4 | After loop or on exception | Close JDBC resources and file handler. | Clean shutdown; log file contains execution trace. |

# Integrations
- **Hive/Impala cluster** – accessed via JDBC; tables referenced are defined in the properties file (`dbname`, source and destination table names).
- **`hive_jdbc_connection` utility** – centralizes connection handling and resource cleanup.
- **Other aggregation jobs** – this utility is typically invoked after raw traffic ingestion and before downstream billing/reporting pipelines; its output tables are consumed by downstream processes (e.g., billing, analytics).

# Operational Risks
- **Unbounded partition list** – large files may cause out‑of‑memory `ArrayList`. *Mitigation*: stream partitions or impose size limits.
- **SQL injection via partition values** – partition dates are concatenated directly into SQL. *Mitigation*: validate date format (`YYYY-MM-DD`) before use.
- **Hard‑coded Hive settings** – dynamic partition mode and truncate statements are static; changes in Hive configuration may break the job. *Mitigation*: externalize these settings to the properties file.
- **No retry logic** – a single failed partition aborts the entire run. *Mitigation*: implement per‑partition try/catch and continue, or schedule retries.
- **FileHandler not rotated by size** – only appends to a single file; log growth may exhaust disk. *Mitigation*: configure `FileHandler` with size limit and count.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.aggregation.traffic_aggr_dups_remove \
    /path/to/partitions.txt \
    /path/to/hive_connection.properties \
    hourly \
    /var/log/traffic_aggr_dups_remove.log
```
- Replace `hourly` with `daily` to run daily aggregation.
- Ensure the properties file contains keys: `HIVE_JDBC_PORT`, `HIVE_HOST`, `dbname`, `traffic_details_daily_tblname`, `traffic_details_aggr_hourly_inter_tblname`, `traffic_details_aggr_hourly_tblname`, `traffic_details_aggr_daily_inter_tblname`.

# Configuration
- **Properties file (required)**  
  - `HIVE_JDBC_PORT` – Hive server port.  
  - `HIVE_HOST` – Hive server hostname.  
  - `dbname` – Hive database name.  
  - Table name keys (as above) for source/destination tables.  

- **Environment** – Java 8+, Hive JDBC driver on classpath, network access to Hive server.

# Improvements
1. **Parameter validation & safe SQL** – validate partition date format and use prepared statements or parameterized queries to eliminate injection risk.  
2. **Partition streaming & retry** – replace `ArrayList` with a streaming iterator and add per‑partition error handling with optional retry/back‑off to improve resilience and memory footprint.