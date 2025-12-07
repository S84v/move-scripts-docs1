# Summary
`table_statistics_calldate_aggregation` is a command‑line Java utility that reads a list of partition dates, queries a Hive/Impala table for aggregated usage metrics (MO/MT SMS, voice minutes, data usage, user counts) per date, formats the results as a semicolon‑delimited record set, and writes them to a specified output file. Execution details are logged to a rotating file handler.

# Key Components
- **class `table_statistics_calldate_aggregation`**
  - `main(String[] args)`: orchestrates argument parsing, configuration loading, file I/O, JDBC connection, query execution, result formatting, and cleanup.
  - `catchException(Exception, Connection, Statement, BufferedWriter)`: logs the exception, closes resources, and exits with status 1.
  - `closeAll(Connection, Statement, BufferedWriter)`: delegates to `hive_jdbc_connection` helpers to close JDBC objects and the logger file handler, then closes the writer.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | `args[0]` – path to properties file | Load Hive/Impala connection parameters and table names | In‑memory `Properties` |
| 2 | `args[1]` – path to file containing partition dates | Read each line into `partitionDates` list | List\<String\> |
| 3 | `args[2]` – output file path | Open `BufferedWriter` for result CSV | File on local FS |
| 4 | `args[3]` – log file path | Initialise `FileHandler` and attach to `logger` | Rotating log file |
| 5 | For each `event_date` in `partitionDates` | Execute aggregated SELECT on `${dbname}.${usage_trend_daily_aggr_tblname}` via Impala JDBC | `ResultSet` with summed columns |
| 6 | ResultSet values | Parse, round to 4 decimal places, compute totals (`total_min`, `total_sms`) | Formatted string appended to `result_buffer` |
| 7 | After loop | Write `result_buffer` to output file | Persisted semicolon‑delimited rows |
| 8 | Finally | Close JDBC resources, logger handler, writer | Clean shutdown |

# Integrations
- **Hive/Impala**: Uses `com.tcl.hive.jdbc.hive_jdbc_connection` to obtain an Impala JDBC connection and to close resources.
- **Properties file**: Supplies DB name, table name, host/port for Hive and Impala.
- **External scripts**: Typically invoked by a scheduler (e.g., Oozie, Airflow) that supplies the four command‑line arguments.
- **Logging**: Writes to a rotating file, enabling downstream log aggregation tools.

# Operational Risks
- **Missing or malformed arguments** → `ArrayIndexOutOfBoundsException`. Mitigation: validate `args.length` before use.
- **Large partition list** → memory pressure from loading all dates into a list. Mitigation: stream dates line‑by‑line.
- **Null numeric fields** → `NumberFormatException` if non‑numeric strings appear. Mitigation: add defensive parsing with fallback defaults.
- **Resource leaks on partial failure** → `closeAll` may be called with null objects. Mitigation: null‑check before closing (already handled by helper methods).
- **Hard‑coded rounding precision** → may not meet future reporting requirements. Mitigation: externalize `places` to config.

# Usage
```bash
java -cp your.jar com.tcl.mnass.aggregation.table_statistics_calldate_aggregation \
    /path/to/usage_trend_aggregation.properties \
    /path/to/partition_dates.txt \
    /path/to/output.csv \
    /path/to/logfile.log
```
- `partition_dates.txt` – one `yyyy-MM-dd` value per line.
- Verify that the properties file contains keys: `usage_trend_daily_aggr_tblname`, `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`.

# Configuration
- **Properties file** (referenced by `args[0]`):
  - `usage_trend_daily_aggr_tblname` – source Hive table.
  - `dbname` – Hive database name.
  - `HIVE_JDBC_PORT`, `HIVE_HOST` – optional (unused in current code).
  - `IMPALAD_JDBC_PORT`, `IMPALAD_HOST` – Impala connection details.
- No environment variables are read directly.

# Improvements
1. **Argument validation & usage help** – add a method to check `args.length` and print expected syntax.
2. **Stream partition dates** – replace `List<String>` with a line‑by‑line processing loop to reduce memory footprint and allow early exit on error.