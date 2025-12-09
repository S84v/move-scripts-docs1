**# Summary**  
`myrep_file_statistics` is a command‑line Java utility that connects to Impala (via Hive JDBC), queries a metadata table (`myrep_file_record_count_tblname`) for a given month, aggregates per‑day file‑record counts for the SNG country across five processes (Traffic, Actives, Activations, Tolling), prints a formatted report to a user‑specified output file, and logs all activity. Any exception forces a clean shutdown with exit code 1.

**# Key Components**  

- `public static void main(String[] args)` – orchestrates configuration loading, DB connection, report generation, and resource cleanup.  
- `printRow(String, Integer, Integer, Integer, Integer)` – formats a single report line to `System.out`.  
- `catchException(Exception, Connection, Statement)` – logs the exception, closes resources, and exits with status 1.  
- `closeAll(Connection, Statement)` – delegates to `hive_jdbc_connection` helpers for statement, connection, and file‑handler cleanup.  

**# Data Flow**  

| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | Properties file (path = `args[0]`) | Load Hive/Impala host, ports, DB name, target table name | In‑memory configuration |
| 2 | Date string (format `yyyy-MM-dd`, `args[1]`) | Parse to `Date`, derive month’s max day | Calendar for day iteration |
| 3 | Output file path (`args[2]`) | Redirect `System.out` to a `PrintStream` writing the report | Report file |
| 4 | Log file path (`args[3]`) | Initialise `java.util.logging.FileHandler` | Log file |
| 5 | Impala JDBC connection (via `hive_jdbc_connection.getImpalaJDBCConnection`) | Create `Statement` | DB connection |
| 6 | For each day 1…max_day: <br>‑ Build CTE query aggregating counts per process | Execute query, read `ResultSet` | Per‑day counts |
| 7 | `printRow` for each day | Write formatted line to report | Report file |
| 8 | After loop, print total row | Write final aggregate | Report file |
| 9 | `finally` block | Close DB resources, file handler | Clean shutdown |

**# Integrations**  

- **Impala/Hive** – accessed through `com.tcl.hive.jdbc.hive_jdbc_connection`. The utility runs read‑only `SELECT` statements against the metadata table.  
- **Logging subsystem** – uses Java Util Logging; log file path supplied by caller.  
- **External scripts** – typically invoked by a scheduler (e.g., Oozie, Airflow) after daily ingestion jobs to produce a monthly statistics report. The output file may be consumed by downstream reporting pipelines or emailed to operations.

**# Operational Risks**  

- **Incorrect date range** – If `date_value` is not the first day of a month, `max_date` may be wrong, leading to missing or extra rows. *Mitigation*: validate that the day component equals `01` before processing.  
- **Null result sets** – The query assumes all four CTEs return a row; missing data yields `null` values that are handled but may hide data quality issues. *Mitigation*: add sanity checks and alert on zero counts.  
- **Resource leakage on abrupt termination** – Although `closeAll` is called in `finally`, an `System.exit(1)` inside `catchException` bypasses the `finally` block. *Mitigation*: move cleanup before `System.exit` or use a shutdown hook.  
- **Hard‑coded country code (`SNG`)** – Limits reuse for other markets. *Mitigation*: externalize as a property.  

**# Usage**  

```bash
# Compile (if not already packaged)
javac -cp path/to/hive-jdbc.jar:path/to/impala-jdbc.jar com/tcl/mnass/validation/myrep_file_statistics.java

# Run
java -cp .:path/to/hive-jdbc.jar:path/to/impala-jdbc.jar \
     com.tcl.mnass.validation.myrep_file_statistics \
     /path/to/config.properties \
     2024-07-01 \
     /tmp/myrep_report_2024_07.txt \
     /var/log/myrep_file_statistics.log
```

*Debug*: set `logger.setLevel(Level.FINE)` or add `-Djava.util.logging.config.file=logging.properties` to increase verbosity.

**# Configuration**  

The properties file (first argument) must contain at least:

| Property | Description |
|----------|-------------|
| `HIVE_HOST` | Impala/Hive server hostname |
| `HIVE_JDBC_PORT` | Hive JDBC port (unused in current code) |
| `IMPALAD_HOST` | Impala daemon hostname |
| `IMPALAD_JDBC_PORT` | Impala JDBC port |
| `dbname` | Target database name |
| `myrep_file_record_count_tblname` | Table storing per‑file record counts |

No environment variables are read directly; all configuration is property‑driven.

**# Improvements**  

1. **Parameterise country code** – add a property (e.g., `country_code=SNG`) and replace hard‑coded literals in the query.  
2. **Refactor query construction** – use `PreparedStatement` with bind variables for the date to avoid string concatenation and improve readability/maintainability.