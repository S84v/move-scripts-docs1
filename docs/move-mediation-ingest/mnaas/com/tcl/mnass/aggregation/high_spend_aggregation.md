# Summary
`high_spend_aggregation` is a command‑line Java utility that, for each partition date supplied, queries raw traffic Hive/Impala tables to compute per‑subscriber high‑spend metrics (MO/MT SMS counts, voice minutes, total data usage). It merges results on the composite key *(tcl_secs_id, serv_abbr, bu_prompt, msisdn)*, derives totals, formats a semicolon‑delimited record set, and writes the output to a file. Execution is logged to a rotating file handler.

# Key Components
- **class `high_spend_aggregation`** – entry point; orchestrates configuration loading, DB connection, query execution, result merging, and file output.  
- **`main(String[] args)`** – parses arguments, loads properties, reads partition list, opens Impala JDBC connection, iterates over dates, runs six analytical queries, builds hash maps, computes derived fields, writes result file, handles exceptions.  
- **`catchException(Exception, Connection, Statement, BufferedWriter)`** – logs exception, closes resources, exits with error code.  
- **`closeAll(Connection, Statement, BufferedWriter)`** – delegates to `hive_jdbc_connection` utility to close JDBC objects and file handler, then closes the writer.  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Config load | `args[0]` – path to properties file | `Properties.load()` | In‑memory config map (`high_spend_aggregation`) |
| Partition list | `args[1]` – file containing partition dates (one per line) | BufferedReader → `List<String> partitionDates` | List of dates to process |
| Output file | `args[2]` – target CSV path | `BufferedWriter` writes aggregated rows | Semicolon‑delimited report |
| Log file | `args[3]` – log file path | `FileHandler` attached to `logger` | Execution log |
| DB connection | Hive/Impala host/port from properties | `hive_jdbc_connection.getImpalaJDBCConnection()` | `java.sql.Connection` |
| Queries per date | Six SQL strings built with `event_date` | `Statement.executeQuery()` → `ResultSet` → populate hash maps (`mo_sms_q_hm`, `mt_sms_q_hm`, `mo_voice_q_hm`, `mt_voice_q_hm`, `total_usage_q_hm`) | In‑memory maps keyed by composite key |
| Merge & compute | Iterate over `distinct_key_hm` | Parse key, fetch values from maps, convert units, round to 4 decimal places, build output line | Appended to `result_buffer` then written to output file |
| Cleanup | – | `closeAll()` in finally block | JDBC resources and file handles released |

# Integrations
- **Hive/Impala** – accessed via custom JDBC wrapper `com.tcl.hive.jdbc.hive_jdbc_connection`. Queries target tables defined in properties: `traffic_details_raw_daily_with_no_dups_tblname`, `high_spend_inter_daily_aggr_tblname`, `high_spend_daily_aggr_tblname`.  
- **External scripts** – typically invoked from a shell/automation job that supplies the four arguments (properties file, partition list, output CSV, log file).  
- **Logging** – uses Java Util Logging; log file path is externalized via argument.  

# Operational Risks
- **Unbounded memory usage** – hash maps hold all distinct keys for a partition date; large partitions may cause OOM. *Mitigation*: stream results per key or process in batches.  
- **Hard‑coded rounding scale** (4 decimal places) may not meet future precision requirements. *Mitigation*: externalize `places` to config.  
- **No input validation** for command‑line arguments; missing or malformed files cause immediate failure. *Mitigation*: add pre‑flight checks and user‑friendly error messages.  
- **SQL injection risk** – date values are concatenated directly into queries. *Mitigation*: use prepared statements with parameter binding.  
- **Resource leak on exception before writer initialization** – `bw` may be null when `catchException` calls `closeAll`. *Mitigation*: null‑check before closing.  

# Usage
```bash
java -cp high_spend_aggregation.jar:lib/* \
  com.tcl.mnass.aggregation.high_spend_aggregation \
  /path/to/high_spend_aggregation.properties \
  /path/to/partition_dates.txt \
  /output/high_spend_report.csv \
  /logs/high_spend_aggregation.log
```
- `high_spend_aggregation.properties` contains DB hosts, ports, table names, and `dbname`.  
- `partition_dates.txt` – one `yyyy-MM-dd` per line.  

# Configuration
- **Properties file (arg 0)**  
  - `traffic_details_raw_daily_with_no_dups_tblname`  
  - `high_spend_inter_daily_aggr_tblname` (currently unused)  
  - `high_spend_daily_aggr_tblname` (currently unused)  
  - `dbname` – Hive database name  
  - `HIVE_JDBC_PORT`, `HIVE_HOST` (unused – Impala used)  
  - `IMPALAD_JDBC_PORT`, `IMPALAD_HOST` – Impala connection details  
- **Static constants**  
  - `places = 4` – decimal precision  
  - Hive dynamic partition settings (defined but not applied).  

# Improvements
1. **Streamline memory usage** – replace the per‑date hash‑map aggregation with a single pass over a unified result set (e.g., using SQL `FULL OUTER JOIN` or `UNION ALL` with aggregation) to avoid loading all keys into memory.  
2. **Parameterize SQL** – refactor query construction to use `PreparedStatement` with bind variables for `event_date` and table names, eliminating concatenation and improving security and plan reuse.