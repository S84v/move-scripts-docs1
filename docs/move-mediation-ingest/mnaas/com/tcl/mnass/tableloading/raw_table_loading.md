# Summary
`raw_table_loading` is a command‑line Java utility that loads raw telecom data from staging Hive tables into target Hive tables. It selects the appropriate ETL logic based on the `processname` argument (e.g., traffic_details, activations, actives, etc.), constructs Hive INSERT … SELECT statements (often via `QueryReader` and YAML query templates), executes them, and optionally runs post‑load rejection handling for SiriusXM records.

# Key Components
- **Class `raw_table_loading`**
  - `main(String[] args)`: parses CLI arguments, configures logging, establishes Hive JDBC connection, selects processing branch, builds and executes Hive queries, handles SiriusXM reject/reprocess logic, and ensures cleanup.
  - `catchException(Exception, Connection, Statement)`: logs exception, closes resources, exits with error.
  - `closeAll(Connection, Statement)`: delegates resource cleanup to `hive_jdbc_connection`.
- **Static fields**
  - `logger`, `fileHandler`, `simpleFormatter`: Java Util Logging setup.
  - Hive connection parameters (`HIVE_HOST`, `HIVE_JDBC_PORT`) and dynamic partition settings.
- **External helper classes**
  - `hive_jdbc_connection`: provides `getJDBCConnection`, `closeStatement`, `closeConnection`, `closeFileHandler`.
  - `QueryReader`: reads YAML (`query.yml`) and generates process‑specific Hive SQL.
  - `SiriusXMRejectRecords`: loads and optionally reprocesses reject records for the traffic_details process.

# Data Flow
| Stage | Input | Transformation | Output / Side Effect |
|-------|--------|----------------|----------------------|
| CLI | args[0‑6] (process name, DB, source/dest tables, Hive host/port, log file) | – | Populates local variables, configures logger |
| Config file | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/query.yml` | `QueryReader` parses YAML to produce Hive INSERT statements | Hive SQL strings |
| Hive JDBC | Connection to Hive server (`HIVE_HOST:HIVE_JDBC_PORT`) | Executes `SET` commands for dynamic partitions, then `INSERT … SELECT` statements | Data moved from source to destination Hive tables, partitioned by derived date |
| Optional SiriusXM reject handling | `SiriusXMRejectRecords` reads reject table via same Hive connection | Loads reject records; if `processFlag` true, reprocesses within date range | Updated reject tables, possible additional INSERT/UPDATE |
| Logging | Log file path (args[6]) | Writes INFO/ERROR entries | Audit trail for job execution |
| Cleanup | – | Calls `closeAll` to close JDBC resources and file handler | No lingering connections or file handles |

# Integrations
- **Hive Metastore / HiveServer2**: accessed via `hive_jdbc_connection` for all SQL execution.
- **YAML query repository** (`query.yml`): provides process‑specific SQL templates.
- **SiriusXMRejectRecords**: another internal component invoked only for `traffic_details`.
- **External tables referenced in generated SQL**: e.g., `mnaas.secsid_parent_child_mapping`, `mnaas.kyc_snapshot`, `mnaas.moveconfig`, `feed_customer_secs_mapping`. These are Hive tables populated by upstream ingestion pipelines.

# Operational Risks
- **Hard‑coded file paths** (`/app/hadoop_users/.../query.yml`) – may break if environment changes. *Mitigation*: externalize path via env var or command‑line argument.
- **String concatenation for SQL** – susceptible to injection if arguments are malformed. *Mitigation*: validate inputs; prefer prepared statements where possible.
- **Single point of failure** – the process exits on any exception, potentially leaving downstream tables partially loaded. *Mitigation*: implement transactional staging (e.g., write to temp table then swap) or add retry logic.
- **Dynamic partition settings applied globally** – may affect concurrent jobs on the same Hive session. *Mitigation*: use separate connections per job (already done) and ensure session isolation.
- **Logging file handler not closed on abnormal termination** – could leak file descriptors. *Mitigation*: add shutdown hook to close handler.

# Usage
```bash
# Compile (assuming Hadoop/Hive client jars on classpath)
javac -cp $(hadoop classpath):$(hive --service cli -e 'set hive.metastore.uris;') \
    com/tcl/mnass/tableloading/raw_table_loading.java

# Run
java -cp .:$(hadoop classpath) com.tcl.mnass.tableloading.raw_table_loading \
    <processname> <dbName> <src_table> <dest_table> <hive_host> <hive_port> <log_file> \
    [<flag> <startDate> <endDate>]
# Example for traffic_details with reprocess flag:
java -cp .:$(hadoop classpath) com.tcl.mnass.tableloading.raw_table_loading \
    traffic_details mnaas traffic_inter_raw_daily traffic_raw_daily 10.133.43.97 10000 /var/log/raw_load.log true 2025-01-01 2025-01-31
```
Debugging: increase logger level, inspect generated SQL via log entries, or run the printed query directly in Hive CLI.

# Configuration
- **YAML query definitions**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/query.yml`
- **Hive connection**: `HIVE_HOST` and `HIVE_JDBC_PORT` supplied as CLI args.
- **Logging**: file path provided as CLI arg; uses `java.util.logging` with `SimpleFormatter`.
- **Dynamic partition mode**: hard‑coded `set hive.exec.dynamic.partition.mode=nonstrict` and `set hive.exec.max.dynamic.partitions.pernode=1000`.

# Improvements
1. **Externalize all file paths and Hive settings** (YAML path, dynamic partition configs) to a properties file or environment variables to improve portability across environments.
2. **Refactor process branching**: replace the long `if‑else` chain with a strategy pattern or map of `processname → QueryBuilder` objects to simplify maintenance and enable unit testing of individual query generators.