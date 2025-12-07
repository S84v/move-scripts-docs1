# Summary
`api_med_subs_activity_temp` is a command‑line Java utility that truncates a target Hive/Impala table (`api_med_subs_activity_temp_tblname`) and populates it with aggregated medication subscription activity metrics by executing a single INSERT‑SELECT statement against a set of intermediate tables. It runs via an Impala JDBC connection, logs all actions to a supplied log file, and exits with status 0 on success or 1 on failure.

# Key Components
- **`public static void main(String[] args)`** – Parses arguments, loads JDBC properties, configures logger, establishes Impala connection, truncates target table, runs the INSERT‑SELECT query, and handles exceptions.
- **`catchException(Exception, Connection, Statement)`** – Logs the exception, closes resources, and terminates the JVM with exit code 1.
- **`closeAll(Connection, Statement)`** – Delegates to `hive_jdbc_connection` helpers to close `Statement`, `Connection`, and `FileHandler`.
- **Static logger & `FileHandler`** – Writes operational logs to the file path supplied as `args[1]`.
- **Static configuration strings** – Hive/Impala connection parameters and session settings (dynamic partition mode, execution engine).

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1 | JDBC properties file (`args[0]`) | Loads DB name, host/port for Hive & Impala, and table names. | In‑memory configuration map. |
| 2 | Log file path (`args[1]`) | Initializes `FileHandler` and attaches to `logger`. | Log file created/appended. |
| 3 | Impala JDBC connection | Obtains `Connection` via `hive_jdbc_connection.getImpalaJDBCConnection`. | Live DB session. |
| 4 | Target table name (`api_med_subs_activity_temp_tblname`) | Executes `TRUNCATE TABLE dbname.table`. | Table emptied. |
| 5 | Intermediate tables (`api_med_total_sim_final`, `api_med_mtd_sim_final`, `api_med_30d_sim_final`, `api_med_7d_sim_final`, `api_med_1d_sim_final`) | Runs a single `INSERT INTO … SELECT … LEFT OUTER JOIN …` aggregating activity counts per `buid`. | Populated target table with columns: `activity_1d`, `activity_7d`, `activity_30d`, `activity_mtd`, `total_count`, `buid`. |
| 6 | Exceptions | Caught, logged, resources closed, JVM exits with code 1. | No partial commit (Impala auto‑commits per statement). |

# Integrations
- **`hive_jdbc_connection`** – Utility class providing Impala JDBC connection and resource‑cleanup helpers.
- **Intermediate Hive/Impala tables** – Read‑only sources for activity metrics; must be present in the `mnaas` database.
- **Target Hive/Impala table** – Destination for the aggregated data; part of the production data pipeline for medication subscription analytics.
- **Logging subsystem** – Writes to a file defined at runtime; can be consumed by external monitoring/log aggregation tools.

# Operational Risks
- **Missing/invalid properties file** – Causes `FileNotFoundException`; mitigated by validating argument count and file existence before loading.
- **Impala connection failure** – Leads to immediate abort; mitigate with retry logic or connection pool wrapper.
- **Schema drift in source tables** – `SELECT` column list may break; add defensive schema validation or versioned view.
- **Unbounded truncation** – If the target table is mis‑identified, data loss occurs; enforce table name whitelist.
- **No transaction rollback** – Truncate is irreversible; consider using `INSERT OVERWRITE` if supported.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.tableloading.api_med_subs_activity_temp \
    /path/to/jdbc_properties.conf \
    /var/log/mnaas/api_med_subs_activity_temp.log
```
- Ensure the classpath includes compiled classes and required JDBC driver JARs.
- Verify that the properties file defines `dbname`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`, `api_med_subs_activity_temp_tblname`, and `traffic_details_daily_tblname`.

# Configuration
- **Properties file (first argument)** – Keys:
  - `dbname`
  - `IMPALAD_HOST`
  - `IMPALAD_JDBC_PORT`
  - `api_med_subs_activity_temp_tblname`
  - `traffic_details_daily_tblname` (currently unused)
- **Static session settings** (hard‑coded):
  - `hive.exec.dynamic.partition.mode=nonstrict`
  - `hive.exec.max.dynamic.partitions.pernode=1000`
  - `hive.execution.engine=spark`
- **Logger** – Writes to file path supplied as second argument; uses `java.util.logging.SimpleFormatter`.

# Improvements
1. **Add argument validation and usage help** – Prevents runtime `ArrayIndexOutOfBoundsException` when arguments are missing.
2. **Replace hard‑coded session statements with configurable options** – Allow toggling Spark engine or dynamic partition mode via properties, and execute them before the truncate/insert statements.