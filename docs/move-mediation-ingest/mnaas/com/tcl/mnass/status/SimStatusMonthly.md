# Summary
`SimStatusMonthly` is a command‑line Java utility that loads monthly SIM status data from an intermediate Hive table (`actives_daily_tblname`) into a destination Hive table (`sim_status_monthly_tblname`). For each month listed in a partition‑date file it drops the existing partition, then inserts active and non‑active SIM records for both HOL and SNG sources using a series of prepared Hive INSERT‑SELECT statements. Execution details are logged to a supplied log file.

# Key Components
- **class `SimStatusMonthly`** – entry point; parses arguments, loads properties, drives the monthly load process.  
- **`main(String[] args)`** – validates arguments, configures logger, reads partition list, invokes `updateSimStatusMonthlyTable`.  
- **`updateSimStatusMonthlyTable(List<String> partitionDates)`** – core logic: builds and executes Hive DDL/DML for each month, handling date calculations and prepared statements.  
- **SQL query strings** – `alterQuery`, `holActiveSIMQuery`, `sngActiveQuery`, `holNonActiveSIMQuery`, `sngNonActiveQuery`; parameterised Hive SQL used for partition management and data insertion.  
- **Utility `getStackTrace(Exception)`** – converts exception stack trace to string for logging.  
- **Logging** – `java.util.logging.Logger` with `FileHandler` and `SimpleFormatter`.  
- **JDBC helper** – `hive_jdbc_connection` provides Impala JDBC connection and resource‑cleanup methods.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1. Argument parsing | args[0]=properties file, args[1]=process name, args[2]=partition list file, args[3]=log file | Load JDBC properties, configure logger | None |
| 2. Partition list read | Text file with `yyyy-MM-dd` lines | Populate `List<String> partitionDates` | In‑memory list |
| 3. For each partitionDate | `partitionDate` string | - Compute month start, end, next month dates<br>- Prepare and execute `ALTER TABLE … DROP PARTITION`<br>- Execute four INSERT‑SELECT statements (HOL active, SNG active, HOL non‑active, SNG non‑active) with bound date parameters | Rows inserted into `dbName.destTable` partition `partition_date_month = partitionDate` |
| 4. Logging | Logger writes to `args[3]` | Status, errors, stack traces | Log file |
| 5. Resource cleanup | JDBC `Connection`, `Statement`, `PreparedStatement`, `FileHandler` | Closed in error path; normal exit relies on JVM shutdown | No lingering resources |

External services:
- Impala/Hive cluster accessed via JDBC (`impalaHost:impalaPort`).
- File system for properties, partition list, and log file.

# Integrations
- **`hive_jdbc_connection`** – shared library used across move‑mediation utilities for obtaining Impala connections and safe closure of JDBC resources.  
- **Other move‑mediation scripts** – follows the same command‑line pattern as `usage_trend_aggregation`, `usage_trend_loading`, `SimInventoryStatusMonthly`; can be orchestrated by the same batch scheduler (e.g., Oozie, Airflow) that supplies the arguments.  
- **Hive metastore** – interacts with tables defined in the properties file (`actives_daily_tblname`, `sim_status_monthly_tblname`).  

# Operational Risks
- **Incorrect partition format** – `ParseException` aborts the whole run. *Mitigation*: validate partition file before execution; enforce `yyyy-MM-dd` pattern.  
- **Partial load on failure** – if a statement fails, earlier inserts for the same month remain, leading to inconsistent data. *Mitigation*: wrap month processing in a Hive transaction (if supported) or add a cleanup step to drop the partition on error.  
- **Hard‑coded SIM ID exclusion (`tcl_secs_id <> 37226`)** – future schema changes may require updates. *Mitigation*: externalise exclusion list to config.  
- **Resource leakage on success path** – statements and connection are not explicitly closed after successful loop. *Mitigation*: add finally block to close all JDBC resources and the file handler.  
- **Logging to a single file without rotation** – log file may grow indefinitely. *Mitigation*: configure `FileHandler` with size limit and count.

# Usage
```bash
java -cp move-mediation-ingest.jar \
     com.tcl.mnass.status.SimStatusMonthly \
     /path/to/jdbc.properties \
     sim_status_monthly \
     /path/to/partition_dates.txt \
     /var/log/sim_status_monthly.log
```
- `args[0]` – JDBC properties file (keys: `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`, `dbname`, `actives_daily_tblname`, `sim_status_monthly_tblname`).  
- `args[1]` – must be exactly `sim_status_monthly`.  
- `args[2]` – file containing one `yyyy-MM-dd` date per line.  
- `args[3]` – path to log file (appended on each run).

# Configuration
- **JDBC properties file** (example):
  ```
  IMPALAD_JDBC_PORT=21050
  IMPALAD_HOST=impala.example.com
  dbname=telecom_dw
  actives_daily_tblname=actives_raw_daily
  sim_status_monthly_tblname=sim_status_monthly
  ```
- No environment variables are read; all configuration is supplied via the properties file and command‑line arguments.

# Improvements
1. **Resource Management** – implement a `try‑with‑resources` or explicit `finally` block to close `Connection`, all `PreparedStatement`s, `Statement`, and `FileHandler` after successful execution.  
2. **Transactional Safety** – add logic to drop the target partition on error and/or use Hive’s `INSERT OVERWRITE` semantics to guarantee atomic month‑level loads.