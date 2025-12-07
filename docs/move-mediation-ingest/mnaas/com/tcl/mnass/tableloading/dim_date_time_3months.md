# Summary
`dim_date_time_3months` is a command‑line Java utility that truncates a Hive dimension table (`dim_date_time_3months_tblname`) and repopulates it with calendar‑date, time, and timestamp rows covering the last 90 days. It runs via a Hive JDBC connection, logs actions to a supplied log file, and exits with status 0 on success or 1 on failure.

# Key Components
- **`public static void main(String[] args)`** – Entry point; parses arguments, loads properties, establishes Hive JDBC connection, executes truncate and insert‑select logic, handles exceptions, and closes resources.  
- **`catchException(Exception, Connection, Statement)`** – Logs the exception, closes resources, and forces process exit with code 1.  
- **`closeAll(Connection, Statement)`** – Delegates to `hive_jdbc_connection` utility to close statement, connection, and file handler.  
- **Static logger & file handler** – Configured to write detailed logs to the user‑provided log file.  
- **SQL strings** – `hiveDynamicPartitionMode`, `hiveDynamicPartitionPerNode`, and the main `INSERT … SELECT` CTE that builds the 90‑day calendar.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1 | Property file path (`args[0]`), process name (`args[1]`), log file path (`args[2]`) | Load JDBC host/port, DB name, target table name | In‑memory configuration |
| 2 | Hive JDBC connection | Execute `SET` statements for dynamic partitioning | Session configuration |
| 3 | Target table name (`dim_date_time_3months_tblname`) | `TRUNCATE TABLE db.table` | Table emptied |
| 4 | Current date (system) | Compute `today` and `newDate = today - 90 days` | Date range strings |
| 5 | Hive tables `mnaas.dim_date` and `mnaas.dim_time` | `INSERT INTO db.table PARTITION(calendar_date) SELECT …` | Populated rows for each date‑time combination (90 days × distinct times) |
| 6 | Logger & file handler | Write informational and error logs | Log file on filesystem |

External services: Hive server (via JDBC). No queues or other services.

# Integrations
- **`hive_jdbc_connection`** – Custom wrapper used to obtain and close Hive JDBC connections and resources.  
- **`mnaas.dim_date` & `mnaas.dim_time`** – Source dimension tables that must exist in the Hive metastore.  
- **Production orchestration** – Typically invoked by a scheduler (e.g., Oozie, Airflow, cron) that supplies the property file and log path.

# Operational Risks
- **Hard‑coded 90‑day window** – May become stale if business requirements change; requires code change.  
- **No validation of property values** – Missing or malformed properties cause `NullPointerException` before logging. Mitigation: add pre‑flight checks.  
- **Single‑threaded execution** – Concurrent runs could conflict on the same target table. Mitigation: enforce lock or schedule non‑overlapping runs.  
- **Truncate without backup** – Data loss if insert fails after truncate. Mitigation: use `INSERT OVERWRITE` or stage into a temp table then swap.  
- **JDBC connection timeout not configured** – Long runs may hang. Mitigation: set appropriate timeout properties in the connection string.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.tableloading.dim_date_time_3months \
    /path/to/config.properties \
    dim_date_time_3months \
    /var/log/dim_date_time_3months.log
```
- Ensure the classpath includes the compiled classes and required Hive JDBC driver JARs.  
- Verify that the Hive server is reachable from the host running the utility.

# Configuration
Properties file (referenced by `args[0]`) must contain:
- `dbname` – Hive database name.  
- `HIVE_JDBC_PORT` – Port for HiveServer2.  
- `HIVE_HOST` – Hostname or IP of HiveServer2.  
- `IMPALAD_JDBC_PORT` – (unused in current code).  
- `IMPALAD_HOST` – (unused).  
- `dim_date_time_3months_tblname` – Target table to truncate and load.

# Improvements
1. **Add transactional safety** – Replace truncate + insert with `INSERT OVERWRITE` or load into a staging table and rename to avoid data loss on partial failure.  
2. **Parameterize date range** – Accept start/end dates as arguments or config entries instead of a fixed 90‑day window, enabling flexible back‑fills.