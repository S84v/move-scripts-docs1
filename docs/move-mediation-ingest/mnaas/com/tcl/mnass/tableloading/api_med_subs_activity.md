# Summary
`api_med_subs_activity` is a command‑line Java utility that loads daily API medication subscription activity data from an intermediate Hive/Impala table (`api_med_subs_activity_temp_tblname`) into a production Hive table (`api_med_subs_activity_tblname`). It truncates the target table, then inserts the data partitioned by `buid`. All operations are performed via an Impala JDBC connection and are logged to a supplied log file.

# Key Components
- **`main(String[] args)`** – Entry point; parses arguments, loads properties, establishes JDBC connection, executes truncate and insert statements, handles exceptions, and ensures cleanup.  
- **`catchException(Exception e, Connection c, Statement s)`** – Centralized error logging, resource cleanup, and process termination.  
- **`closeAll(Connection c, Statement s)`** – Delegates to `hive_jdbc_connection` utilities to close `Statement`, `Connection`, and `FileHandler`.  

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | Properties file (path = `args[0]`) | Load DB name, host/port for Hive & Impala, table names. | In‑memory configuration map. |
| 2 | Log file path (`args[1]`) | Initialise `java.util.logging.FileHandler` and attach to logger. | Log file created/appended. |
| 3 | Impala JDBC connection (via `hive_jdbc_connection.getImpalaJDBCConnection`) | Open network connection to Impala server. | Live `Connection` object. |
| 4 | `truncate table <dbname>.<api_med_subs_activity_tblname>` | Execute DDL to delete all rows in target table. | Target table emptied. |
| 5 | `insert into <dbname>.<api_med_subs_activity_tblname> partition(buid) select … from <dbname>.<api_med_subs_activity_temp_tblname>` | Execute DML to copy rows from temp table into target, partitioned by `buid`. | Target table populated with fresh data. |
| 6 | Exception handling | Log stack trace, close resources, exit with status 1. | Process termination on error. |
| 7 | `finally` block | Close JDBC resources and file handler. | Clean shutdown. |

# Integrations
- **`hive_jdbc_connection`** – Utility class providing Impala/JDBC connection creation and resource‑cleanup methods.  
- **Hive/Impala Metastore** – Source (`api_med_subs_activity_temp_tblname`) and destination (`api_med_subs_activity_tblname`) tables reside in the same Hive database (`dbname`).  
- **Production ETL pipeline** – Typically invoked after upstream jobs populate the temporary table; downstream jobs may read from the final table.  

# Operational Risks
- **Data loss** – Truncation occurs before insert; failure after truncate leaves the target table empty. *Mitigation*: implement a staging table or transactional copy (e.g., `INSERT OVERWRITE`).  
- **No partition validation** – Assumes `buid` column exists and is correctly populated; malformed data could cause insert failures. *Mitigation*: add schema validation or pre‑insert row count checks.  
- **Hard‑coded JDBC hosts/ports** – Changes require property file updates; missing/incorrect values cause connection failures. *Mitigation*: validate properties at startup.  
- **Single‑threaded, no retry** – Transient network issues abort the job. *Mitigation*: incorporate retry logic around connection and statement execution.  

# Usage
```bash
# Compile (if not using pre‑built jar)
javac -cp <classpath> com/tcl/mnass/tableloading/api_med_subs_activity.java

# Run
java -cp <classpath> com.tcl.mnass.tableloading.api_med_subs_activity \
    /path/to/api_med_subs_activity.properties \
    /var/log/mnaas/api_med_subs_activity.log
```
*Replace `<classpath>` with required Hive/Impala JDBC driver jars and project dependencies.*

# Configuration
Properties file referenced by `args[0]` must contain:

| Property | Description |
|----------|-------------|
| `dbname` | Hive database name containing source & target tables. |
| `HIVE_JDBC_PORT` | (Unused in current code) Hive JDBC port. |
| `HIVE_HOST` | (Unused in current code) Hive server host. |
| `IMPALAD_JDBC_PORT` | Impala JDBC port. |
| `IMPALAD_HOST` | Impala server host. |
| `api_med_subs_activity_tblname` | Target table name. |
| `api_med_subs_activity_temp_tblname` | Source temporary table name. |

# Improvements
1. **Atomic Load** – Replace `TRUNCATE` + `INSERT` with `INSERT OVERWRITE` or use a staging table and swap partitions to guarantee that the target table is never left empty on failure.  
2. **Dynamic Partition Settings** – Enable and configure Hive dynamic‑partition parameters (`hive.exec.dynamic.partition.mode`, `hive.exec.max.dynamic.partitions.pernode`) to support future schema changes and improve scalability.  