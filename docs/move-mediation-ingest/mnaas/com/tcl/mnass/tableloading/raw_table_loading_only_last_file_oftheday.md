# Summary
`raw_table_loading_only_last_file_oftheday` is a command‑line Java utility that loads the most recent daily file for a given process (currently only “actives”) from a staging Hive table into a target Hive table. It determines the partition date and country prefix from the source file name, drops any existing partition in the destination, and inserts the data using a Hive INSERT … SELECT statement that varies for HOL or SNG country prefixes.

# Key Components
- **Class `raw_table_loading_only_last_file_oftheday`** – entry point; orchestrates argument parsing, Hive connection, query generation, execution, and cleanup.  
- **`main(String[] args)`** – parses six positional arguments, configures logging, establishes JDBC connection, builds partition queries, selects the appropriate INSERT statement, executes DROP and INSERT, logs progress.  
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, exits with status 1.  
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` helpers to close statement, connection, and file handler.  

# Data Flow
| Step | Input | Processing | Output / Side Effect |
|------|-------|------------|----------------------|
| 1 | CLI args: `processname dbName src_table dest_table hiveHost hivePort logFile` | Stored in local variables. | N/A |
| 2 | Hive JDBC connection (via `hive_jdbc_connection.getJDBCConnection`) | Opens TCP connection to HiveServer2. | `Connection con`. |
| 3 | `partitionDateQuery` & `partitionCntryquery` executed on `dbName.src_table` | Extracts partition date (`yyyy-MM-dd`) and country prefix from source file name. | `partitionDate`, `cntry`. |
| 4 | `drop_query` constructed | Generates `ALTER TABLE … DROP IF EXISTS PARTITION (…)`. | Hive partition removal. |
| 5 | Conditional INSERT query (`query`) built | Two variants:  
- HOL: selects all columns plus extra fields, filters `split(filename,'[_]')[0] = 'HOL'`.  
- SNG: joins `feed_customer_secs_mapping` and filters `= 'SNG'`. | Hive INSERT statement. |
| 6 | Execution of `drop_query`, `hiveDynamicPartitionMode`, `hiveDynamicPartitionPerNode`, then `query` via `Statement.execute`. | Data movement from staging to target table, partitioned by `partition_date` and `country_prefix`. | Target Hive table populated; logs written to `logFile`. |

External services:
- HiveServer2 (JDBC)
- Local filesystem for log file.

# Integrations
- **`hive_jdbc_connection`** – utility class providing static methods for obtaining and closing Hive JDBC connections and resources.  
- **`feed_customer_secs_mapping`** – Hive table used in the SNG branch for a right‑outer join.  
- **Logging** – Java `java.util.logging` writes to the file path supplied as argument 6.  
- **Other move‑mediation utilities** – This class is a specialized variant of the generic `raw_table_loading` tool; invoked by orchestration scripts (e.g., shell or Airflow) that schedule daily loads.

# Operational Risks
- **Hard‑coded process name** – Only “actives” is supported; any other value leads to `null` queries and runtime failure. *Mitigation*: validate `processname` early and exit with clear error.  
- **Assumption of single distinct partitionDate/cntry** – If source contains multiple dates or prefixes, the last row wins, potentially loading wrong partition. *Mitigation*: enforce uniqueness in source or add explicit checks.  
- **String concatenation for SQL** – Susceptible to injection if arguments are malformed. *Mitigation*: sanitize inputs or use prepared statements for dynamic parts.  
- **No retry on transient Hive failures** – Job aborts on first SQLException. *Mitigation*: implement retry logic with exponential back‑off.  
- **Resource leakage on early exit** – `catchException` calls `System.exit(1)` after closing resources, but if logger/fileHandler fails to close, file descriptors may remain. *Mitigation*: ensure `closeAll` is robust and called in a `finally` block (already present).  

# Usage
```bash
java -cp move-mediation-ingest.jar \
    com.tcl.mnass.tableloading.raw_table_loading_only_last_file_oftheday \
    actives mydb staging_actives_tbl target_actives_tbl hive-host.example.com 10000 /var/log/raw_load.log
```
- Ensure Hive JDBC driver JAR is on the classpath.  
- Verify that the source table contains files named with the expected pattern (e.g., `...YYYYMMDD...`).  

# Configuration
- **CLI arguments** (no external config files):
  1. `processname` – currently only `actives`.
  2. `dbName` – Hive database name.
  3. `src_tableName` – staging table.
  4. `dest_tableName` – target table.
  5. `HIVE_HOST` – HiveServer2 host.
  6. `HIVE_JDBC_PORT` – HiveServer2 port.
  7. `logFilePath` – absolute path for the log file.
- **Static Hive settings** applied at runtime:
  - `set hive.exec.dynamic.partition.mode=nonstrict`
  - `set hive.exec.max.dynamic.partitions.pernode=1000`

# Improvements
1. **Parameterize process handling** – Replace the hard‑coded `if (processname.equals("actives"))` block with a strategy pattern or configuration map to support additional processes without code changes.  
2. **Replace string‑concatenated SQL with prepared statements** – Use `PreparedStatement` for the partition queries and INSERT to eliminate injection risk and improve readability.