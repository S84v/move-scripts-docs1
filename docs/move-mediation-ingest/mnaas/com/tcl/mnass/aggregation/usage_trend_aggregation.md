# Summary
`usage_trend_aggregation` is a command‑line Java utility that reads a Hive/Impala JDBC configuration file and a list of partition dates, then for each date executes a series of Hive SELECT queries to retrieve aggregated traffic metrics (SMS, voice, data usage, active users, max usage). The results are combined, transformed (unit conversions, rounding), and written as a delimited text file for downstream consumption.

# Key Components
- **class `usage_trend_aggregation`** – entry point; orchestrates configuration loading, JDBC connection, query execution, result aggregation, and file output.  
- **`main(String[] args)`** – parses arguments, loads properties, reads partition list, opens JDBC/Impala connection, iterates over partitions, builds hash maps of metric values, computes derived fields, writes output.  
- **`catchException(Exception, Connection, Statement, BufferedWriter)`** – logs exception, closes resources, exits with error.  
- **`closeAll(Connection, Statement, BufferedWriter)`** – delegates to `hive_jdbc_connection` helpers to close JDBC objects and file handler, then closes the writer.  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Config load | `args[0]` – properties file | `Properties.load` | JDBC host/port, table names, DB name |
| Partition list | `args[1]` – file path | BufferedReader → `List<String>` | In‑memory list of partition dates |
| Result file | `args[2]` – output file path | `BufferedWriter` | Delimited text rows (one per distinct key) |
| Log file | `args[3]` – log file path | `FileHandler` (rotating) | Execution logs |
| DB connection | Impala host/port from properties | `hive_jdbc_connection.getImpalaJDBCConnection` | `Connection` object |
| Queries per partition | Partition date, table names | 9 SELECT statements → `ResultSet` → populate `Map<String,String>` per metric | Metric hash maps keyed by `tcl_secs_id|serv_abbr|bu_prompt` |
| Aggregation | Metric maps, distinct key set | Numeric parsing, unit conversion (seconds→minutes, bytes→MiB), rounding to 4 decimal places, derived calculations (averages) | String buffer appended to output file |

External services: Impala/Hive cluster accessed via JDBC; local filesystem for config, partition list, output, and log files.

# Integrations
- **`hive_jdbc_connection`** – utility class (not shown) providing JDBC connection creation and safe close methods.  
- **Other aggregation utilities** (`traffic_aggr_adhoc_aggregation`, `traffic_aggr_dups_remove`, `traffic_aggr_loading`) share the same property keys and likely run in the same ETL pipeline, feeding intermediate tables that this script reads.  
- **Downstream consumers** – any process that ingests the generated delimited file (e.g., reporting jobs, data warehouse loaders).

# Operational Risks
- **Resource leakage** – failure to close `ResultSet` objects; mitigated by explicit `close()` after each loop (already present).  
- **Unbounded memory** – all metric maps are kept in memory per partition; large partition cardinality could cause OOM. Mitigation: stream results or batch write per key.  
- **Hard‑coded rounding scale** (4 decimal places) may not meet business precision; review requirement.  
- **Implicit assumption of non‑null keys**; malformed data could produce empty key `" "` leading to spurious map entries – code removes it but may hide data issues.  
- **No retry on JDBC failures**; a transient network glitch aborts the whole run. Add retry logic or checkpointing.

# Usage
```bash
java -cp usage_trend_aggregation.jar:lib/* \
    com.tcl.mnass.aggregation.usage_trend_aggregation \
    /path/to/usage_trend_aggregation.properties \
    /path/to/partition_dates.txt \
    /path/to/output_usage_trend.txt \
    /path/to/usage_trend.log
```
- `partition_dates.txt` – one `yyyy-MM-dd` value per line.  
- Verify that the Impala service is reachable and the tables referenced in the properties exist.

# Configuration
Properties file (example keys):
```
traffic_details_aggr_daily_tblname=traffic_details_aggr_daily
actives_aggr_daily_tblname=actives_aggr_daily
usage_trend_inter_daily_aggr_tblname=usage_trend_inter_daily_aggr
traffic_details_daily_tblname=traffic_details_daily
traffic_details_raw_daily_with_no_dups_tblname=traffic_details_raw_daily_no_dups
dbname=telecom_db
HIVE_JDBC_PORT=10000
HIVE_HOST=hive.example.com
IMPALAD_JDBC_PORT=21050
IMPALAD_HOST=impala.example.com
```
No environment variables are referenced directly; all runtime parameters are passed via command‑line arguments.

# Improvements
1. **Streamline metric collection** – replace per‑metric hash maps with a single `Map<String, MetricRecord>` that aggregates all fields in one pass, reducing memory footprint and lookup overhead.  
2. **Add checkpoint/retry mechanism** – persist the last successfully processed partition date (e.g., to a small state file) and implement retry logic for transient JDBC failures to improve resilience in production.