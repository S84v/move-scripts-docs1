# Summary
`telene_file_stats` is a command‑line Java utility that, for a specified month, queries Impala/Hive tables containing per‑file record counts and file sizes for Telena processes (Traffic, Actives, Activations, Tolling, Siminventory) across HOL and SNG countries. It aggregates daily counts, prints a formatted statistics report to a user‑specified output file, logs execution details, and exits with status 0 on success or 1 on any exception.

# Key Components
- **class `telene_file_stats`** – entry point; contains `main`, helper methods, and static logger.
- **`main(String[] args)`** – parses arguments, loads properties, establishes JDBC connection, drives daily aggregation loops, prints report, handles exceptions.
- **`printRow(String, Integer… )`** – formats a single report line (date or total) to `System.out`.
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, terminates with exit 1.
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` utility for clean shutdown of JDBC objects and log handler.
- **Static logger & `FileHandler`** – writes detailed logs to a rotating file defined by the fourth CLI argument.

# Data Flow
| Phase | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Config load | `args[0]` – path to Java `.properties` file | `Properties.load` → extracts Hive/Impala host/port, DB name, table names | In‑memory configuration |
| 2. Runtime parameters | `args[1]` – target month (`yyyy-MM-dd`), `args[2]` – output report file, `args[3]` – log file path | Parsed into `date_value`, `output_file_name`, `fileHandler` | Opens log file, redirects `System.out` to `output_file_name` |
| 3. DB connection | Impala JDBC URL built from `IMPALAD_HOST`/`IMPALAD_JDBC_PORT` | `hive_jdbc_connection.getImpalaJDBCConnection` → `Connection` | Active Impala session |
| 4. Daily count aggregation | For each day `1..max_date` derived from `date_value` | Executes a massive CTE query joining ten sub‑queries (HOL/SNG per process) → `ResultSet` → extracts integer counts, accumulates totals | Prints one formatted row per day via `printRow` |
| 5. Totals row | After loop | Sums of all daily counters | Printed as “Total” row |
| 6. File‑size statistics (last 5 days) | Same day loop (min_date..max_date) | Executes two separate aggregate queries (HOL and SNG Traffic) → computes average size in KB, rounds to 2 decimals | Prints “Average file size …” lines |
| 7. Cleanup | `finally` block | Calls `closeAll` → closes `Statement`, `Connection`, `FileHandler` | Releases resources |

# Integrations
- **Impala/Hive** – accessed via `com.tcl.hive.jdbc.hive_jdbc_connection`; the utility can switch to Hive by uncommenting the Hive connection line.
- **Logging** – integrates with Java Util Logging; log file path supplied at runtime.
- **Other move‑mediation components** – typically invoked after daily ingestion pipelines to verify file counts; output report may be consumed by downstream monitoring dashboards or alerting scripts.

# Operational Risks
- **Hard‑coded query joins** – any schema change (e.g., missing country or process) causes `SQLException` and job abort. *Mitigation*: externalize column/table names, add defensive checks.
- **Single‑threaded, full‑month scan** – large months may exceed Impala timeout or memory. *Mitigation*: paginate by day or use incremental queries.
- **No null handling for missing sub‑queries** – if a process has zero rows, the join eliminates the entire row, resulting in missing days. *Mitigation*: use `LEFT JOIN` or `COALESCE` in SQL.
- **Unvalidated CLI arguments** – missing or malformed args cause `ArrayIndexOutOfBoundsException`. *Mitigation*: add argument validation and usage help.
- **File descriptor leak** – `PrintStream` not closed. *Mitigation*: close `out` in `finally`.

# Usage
```bash
java -cp telene_file_stats.jar \
     com.tcl.mnass.validation.telene_file_stats \
     /path/to/config.properties \
     2024-03-01 \
     /tmp/telene_report_202403.txt \
     /var/log/telene_file_stats.log
```
- `config.properties` must contain keys: `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`, `dbname`, and each `*_file_record_count_tblname`.
- The program writes the formatted report to the third argument and logs to the fourth.

# Configuration
| Property | Description |
|----------|--------------|
| `HIVE_HOST` / `IMPALAD_HOST` | Hostnames for Hive/Impala services |
| `HIVE_JDBC_PORT` / `IMPALAD_JDBC_PORT` | Corresponding JDBC ports |
| `dbname` | Target database/schema name |
| `traffic_file_record_count_tblname` | Table storing Traffic file metadata |
| `actives_file_record_count_tblname` | Table for Actives metadata |
| `activations_file_record_count_tblname` | Table for Activations metadata |
| `tolling_file_record_count_tblname` | Table for Tolling metadata |
| `siminventory_file_record_count_tblname` | Table for Siminventory metadata |
| `myrep_file_record_count_tblname` | (Unused in current code) |

# Improvements
1. **Refactor SQL generation** – build queries with a query builder or prepared statements; replace the monolithic CTE string with modular components to improve readability and maintainability.
2. **Introduce robust argument parsing** – use a library (e.g., Apache Commons CLI) to validate required parameters, provide `--help`, and enforce correct date format before proceeding.