# Summary
`customer_invoice_summary_estimated` is a Java utility executed in the MOVE‑Mediation ingest pipeline. It connects to Hive via JDBC, optionally drops stale partitions for a target Hive table (`customer_invoice_summary_estimated_tblname`), and inserts data from a staging temporary table (`customer_invoice_summary_estimated_temp_tblname`) into the target table partitioned by `partition_date`. The utility is invoked with a properties file, a partition‑date file path, a log file path, the current day of month, and the previous month’s year‑month string.

# Key Components
- **class `customer_invoice_summary_estimated`** – entry point; orchestrates Hive connection, partition management, and data load.  
- **`main(String[] args)`** – parses arguments, loads properties, configures logger, establishes Hive JDBC connection, executes dynamic‑partition settings, drops partitions (early‑month logic), runs INSERT‑SELECT.  
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, exits with status 1.  
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` helpers for statement, connection, and file‑handler cleanup.  

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| 1 | `args[0]` – path to properties file | Load DB connection parameters, table names, DB name | In‑memory configuration |
| 2 | `args[1]` – partition date string (e.g., `2023-09-01`) | Used in DROP PARTITION and INSERT statements | Hive partition identifier |
| 3 | `args[2]` – log file path | Initialize `FileHandler` for `java.util.logging` | Log file on local FS |
| 4 | `args[3]` – current day of month (`01`‑`31`) | Determines early‑month partition‑drop logic | Conditional DROP |
| 5 | `args[4]` – previous month year‑month (`2023-08`) | Used for early‑month partition‑drop | Conditional DROP |
| 6 | Hive JDBC connection (via `hive_jdbc_connection`) | Execute `SET` statements for dynamic partitions, optional early‑month DROP, mandatory DROP of target partition, then INSERT‑SELECT from temp table | Target Hive table populated with a new partition containing estimated invoice summary rows |

External services:
- Hive Metastore (via JDBC)
- Local filesystem for properties and log files

# Integrations
- **`hive_jdbc_connection`** – utility class providing `getJDBCConnection`, `closeStatement`, `closeConnection`, and `closeFileHandler`.  
- **Downstream** – the populated `customer_invoice_summary_estimated` Hive table is consumed by downstream billing and reporting jobs (e.g., invoice generation, analytics).  
- **Upstream** – the temporary staging table (`customer_invoice_summary_estimated_temp_tblname`) is populated by earlier ETL steps not shown here.

# Operational Risks
- **Hard‑coded Hive settings** – missing or incompatible Hive version may cause failures; mitigate by externalizing settings.  
- **Early‑month partition drop logic** – assumes partitions exist; dropping non‑existent partitions is safe but may mask missing data; add verification.  
- **No retry logic** – a transient Hive outage aborts the job; wrap statements in retry blocks.  
- **Plain JDBC without Kerberos** – credentials not shown; ensure secure handling of Hive authentication.  
- **Unvalidated arguments** – malformed property file or partition string leads to SQL errors; add argument validation.

# Usage
```bash
java -cp <classpath> com.tcl.mnaas.billingviews.customer_invoice_summary_estimated \
    /path/to/customer_invoice_summary_estimated.properties \
    2023-09-01 \
    /var/log/mnaas/customer_invoice_summary_estimated.log \
    15 \
    2023-08
```
- Ensure the classpath includes Hive JDBC driver and `hive_jdbc_connection` classes.  
- For debugging, set logger level to `FINE` in the properties file or modify `logger.setLevel(Level.ALL)`.

# Configuration
- **Properties file** (path supplied as `args[0]`) must define:
  - `dbname`
  - `HIVE_JDBC_PORT`
  - `HIVE_HOST`
  - `IMPALAD_JDBC_PORT` (unused in current code)
  - `IMPALAD_HOST` (unused)
  - `customer_invoice_summary_estimated_tblname`
  - `customer_invoice_summary_estimated_temp_tblname`
- No environment variables are referenced directly.

# Improvements
1. **Externalize Hive session settings** – move `hiveDynamicPartitionMode` and `hiveDynamicPartitionPerNode` to the properties file to allow tuning without code changes.  
2. **Add argument validation and robust error handling** – verify existence of property keys, partition strings, and implement retry logic for transient Hive failures.