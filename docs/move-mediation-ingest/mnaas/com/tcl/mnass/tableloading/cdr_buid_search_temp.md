# Summary
`cdr_buid_search_temp` is a command‑line Java utility that truncates a Hive staging table (`cdr_buid_seacrh_temp_tblname`) and repopulates it with Call Detail Record (CDR) rows for a list of partition dates. For each date read from a supplied file, it inserts records from the source table `traffic_details_daily_tblname` filtered by `partition_date`. All operations are executed via a Hive JDBC connection and logged to a user‑provided log file.

# Key Components
- **`cdr_buid_search_temp` (class)** – Main entry point; orchestrates configuration loading, logging setup, JDBC connection, table truncation, and per‑date INSERT‑SELECT loops.  
- **`main(String[] args)`** – Parses arguments, reads properties file, loads partition dates, performs DB operations, handles exceptions, and ensures resource cleanup.  
- **`catchException(Exception, Connection, Statement)`** – Logs the exception, closes resources, and exits with status 1.  
- **`closeAll(Connection, Statement)`** – Delegates to `hive_jdbc_connection` utility to close `Statement`, `Connection`, and `FileHandler`.  

# Data Flow
| Step | Input | Processing | Output / Side Effect |
|------|-------|------------|----------------------|
| 1 | `args[0]` – path to properties file | Load DB connection parameters, table names, and database name. | In‑memory configuration values. |
| 2 | `args[1]` – path to partition‑date file | Read each line → `partitionDates` list. | List of date strings (e.g., `2023‑09‑01`). |
| 3 | `args[2]` – log file path | Initialise `FileHandler` and `Logger`. | Log entries written to file. |
| 4 | Hive JDBC connection (host/port from properties) | `truncate table <dbname>.<cdr_buid_seacrh_temp_tblname>` | Target staging table emptied. |
| 5 | For each `event_date` in `partitionDates` | Execute INSERT‑SELECT from `<dbname>.<traffic_details_daily_tblname>` where `partition_date = '<event_date>'`. | Rows inserted into staging table, partitioned implicitly by source data. |
| 6 | Exception handling | Log stack trace, close resources, `System.exit(1)`. | Process termination on error. |
| 7 | Finally block | Close JDBC resources and file handler. | Clean shutdown. |

External services:
- Hive server (via JDBC) – executes DDL/DML.
- Local filesystem – reads properties file, partition‑date file, writes log.

# Integrations
- **`hive_jdbc_connection`** – Custom wrapper providing `getJDBCConnection`, `closeStatement`, `closeConnection`, and `closeFileHandler`.  
- **Source table** `traffic_details_daily_tblname` – Populated by upstream ingestion pipelines (not shown).  
- **Target table** `cdr_buid_seacrh_temp_tblname` – Consumed by downstream processes that require refreshed CDR data indexed by BUID (e.g., reporting or analytics jobs).  
- **Properties file** – Shared configuration used by sibling utilities (`cdr_buid_search`, `api_med_subs_activity`, etc.) to maintain consistent DB connection details and table naming.

# Operational Risks
- **Unbounded partition list** – Very large `partitionDates` may cause long runtimes or JDBC timeouts. *Mitigation*: Validate file size, batch inserts, or parallelize per date.  
- **Hard‑coded truncate** – Full table truncate removes all data before any insert succeeds; failure mid‑loop leaves table empty. *Mitigation*: Use transactional staging (e.g., insert into a temporary table then swap) or add retry logic.  
- **SQL injection via partition file** – Dates are concatenated directly into SQL strings. *Mitigation*: Validate date format (`YYYY‑MM‑DD`) before inclusion or use prepared statements.  
- **Resource leakage on abrupt termination** – `System.exit(1)` bypasses normal JVM shutdown hooks. *Mitigation*: Ensure `closeAll` is always invoked (already done) and consider returning error codes instead of exiting.  
- **Static logger configuration** – Multiple concurrent runs may share the same log file leading to interleaved entries. *Mitigation*: Use unique log filenames per execution (e.g., include timestamp).

# Usage
```bash
java -cp <classpath> com.tcl.mnass.tableloading.cdr_buid_search_temp \
    /path/to/config.properties \
    /path/to/partition_dates.txt \
    /var/log/cdr_buid_search_temp.log
```
- `config.properties` must contain keys: `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`, `cdr_buid_seacrh_temp_tblname`, `traffic_details_daily_tblname`.  
- `partition_dates.txt` – one `YYYY-MM-DD` per line.  
- Monitor the log file for progress and errors.

# Configuration
| Property | Description | Example |
|----------|-------------|---------|
| `dbname` | Hive database containing source and target tables. | `telecom_dw` |
| `HIVE_JDBC_PORT` | Port for HiveServer2 JDBC. | `10000` |
| `HIVE_HOST` | Hostname or IP of HiveServer2. | `hive-prod.example.com` |
| `IMPALAD_JDBC_PORT` | (Unused in current code) Impala JDBC port. | `21050` |
| `IMPALAD_HOST` | (Unused) Impala daemon host. | `impala-prod.example.com` |
| `cdr_buid_seacrh_temp_tblname` | Target staging table name. | `cdr_buid_search_temp` |
| `traffic_details_daily_tblname` | Source daily CDR table name. | `traffic_details_daily` |

# Improvements
1. **Replace string concatenation with prepared statements** to eliminate injection risk and improve performance.  
2. **Implement a transactional staging pattern** (e.g., load into a temporary table then atomically rename) to avoid data loss if the process fails after truncation.