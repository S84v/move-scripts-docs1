# Summary
`business_trend_aggregation` aggregates daily telecom business‑trend metrics for a given event date. It reads configuration, connects to Impala/Hive, executes a fixed set of analytical queries, merges the result sets on the composite key (tcl_secs_id, serv_abbr, businessunit), computes derived counters, and writes a semicolon‑delimited report file.

# Key Components
- **class `business_trend_aggregation`** – entry point and orchestrator.  
- **`main(String[] args)`** – parses arguments, loads properties, opens DB connection, runs queries, builds the output, handles exceptions, and closes resources.  
- **`catchException(Exception, Connection, Statement, BufferedWriter)`** – logs the exception, performs cleanup, and exits with status 1.  
- **`closeAll(Connection, Statement, BufferedWriter)`** – delegates to `hive_jdbc_connection` utilities to close JDBC objects and the file handler, then closes the writer.  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Argument parsing | `args[0]` = properties file path, `args[1]` = event date (yyyy‑MM‑dd), `args[2]` = output CSV path, `args[3]` = log file path | Load `Properties`, initialise logger, extract DB/table names | In‑memory configuration map |
| 2. DB connection | `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` from properties | `hive_jdbc_connection.getImpalaJDBCConnection` → `java.sql.Connection` | Open Impala JDBC session |
| 3. Query execution | Hard‑coded SQL strings that reference the event date and table names | `Statement.executeQuery` for each metric; each `ResultSet` streamed into a `Map<String,String>` keyed by `tcl_secs_id|serv_abbr|businessunit` | Multiple hash maps holding raw counts |
| 4. Key consolidation | `distinct_key_hm` populated with all keys from the maps | Iteration over `distinct_key_hm.keySet()` | Unified key set for final aggregation |
| 5. Metric synthesis | For each key, look up values in the maps, parse to `int`, compute derived fields (`total_active_final_value`, `total_deactivated_final_value`, `total_currently_suspended_final_value`) | Build a `StringBuffer` line with 27 fields (including partition date) | Append line to `result_buffer` |
| 6. File write | `result_buffer` after loop | `BufferedWriter.write` | Destination CSV file (`args[2]`) |
| 7. Cleanup | JDBC objects, file handler, writer | `closeAll` → `hive_jdbc_connection` utilities | Resources released, log file closed |

# Integrations
- **`hive_jdbc_connection`** – utility class (outside this file) providing Impala/JDBC connection management and safe close methods.  
- **Downstream consumers** – any batch job or reporting process that reads the generated CSV (e.g., downstream ETL, BI dashboards).  
- **Upstream scheduler** – typically invoked by an Oozie/Airflow/cron job that supplies the properties file, event date, and output locations.  

# Operational Risks
- **Memory pressure** – All query results are loaded into in‑memory hash maps; large partitions may cause OOM. *Mitigation*: stream aggregation or use temporary Hive tables.  
- **Hard‑coded SQL & string concatenation** – Susceptible to syntax errors if table names change; no parameterisation. *Mitigation*: externalise queries or use prepared statements.  
- **Date handling** – Uses `SimpleDateFormat` without timezone; daylight‑saving shifts could mis‑align 7‑day windows. *Mitigation*: enforce UTC and use `java.time` APIs.  
- **Resource leaks on partial failure** – Only `catchException` triggers cleanup; if an exception occurs before `stmt`/`con` are assigned, `closeAll` may receive nulls (handled) but logger/fileHandler may remain open. *Mitigation*: adopt try‑with‑resources for `Connection`, `Statement`, and `BufferedWriter`.  
- **Log file growth** – `FileHandler` opened in append mode without rotation. *Mitigation*: configure size‑based rotation.  

# Usage
```bash
# Compile (assuming classpath includes hive-jdbc jars)
javac -cp ".:/opt/hive/lib/*" move-mediation-ingest/mnaas/com/tcl/mnass/aggregation/business_trend_aggregation.java

# Run
java -cp ".:/opt/hive/lib/*" com.tcl.mnass.aggregation.business_trend_aggregation \
    /path/to/bus_trend_agg.properties \
    2024-11-30 \
    /data/output/bus_trend_20241130.csv \
    /var/log/bus_trend_agg.log
```
*Debug*: set logger level to `FINE` in the properties file or modify `logger.setLevel(Level.FINE)` before execution.

# Configuration
- **Properties file (arg 0)** – required keys:  
  - `actives_daily_tblname`  
  - `actives_aggr_daily_tblname`  
  - `activations_aggr_daily_tblname`  
  - `business_trend_daily_aggr_tblname`  
  - `actives_aggr_daily_tblname_at_eod`  
  - `actives_raw_daily_at_eod_tblname`  
  - `dbname`  
  - `HIVE_JDBC_PORT` / `HIVE_HOST` (unused)  
  - `IMPALAD_JDBC_PORT` / `IMPALAD_HOST` (active)  
- **Environment** – Java 8+, network