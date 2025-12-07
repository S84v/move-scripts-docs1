# Summary
`usage_overage_charge` is a Java utility executed in the MOVE‑Mediation ingest pipeline. It connects to Hive via JDBC, optionally drops a stale partition for the target table `usage_overage_charge_tblname`, and inserts aggregated usage‑overage charge records from the staging table `usage_overage_charge_temp_tblname` into the target table partitioned by the supplied partition date. The job is invoked with a properties file, a partition‑date argument, a log file path, the current day, and the previous month identifier.

# Key Components
- **`usage_overage_charge` (class)** – Main entry point; orchestrates JDBC connection, logging, partition management, and data load.
- **`main(String[] args)`** – Parses arguments, loads properties, configures logger, establishes Hive connection, executes dynamic‑partition settings, performs conditional partition drop, and runs the INSERT‑SELECT.
- **`catchException(Exception, Connection, Statement)`** – Centralized error handling; logs exception, closes resources, exits with status 1.
- **`closeAll(Connection, Statement)`** – Wrapper that delegates to `hive_jdbc_connection` utility for clean shutdown of JDBC objects and file handler.
- **Static configuration strings** – Hive session settings (`hive.exec.dynamic.partition.mode=nonstrict`, `hive.exec.max.dynamic.partitions.pernode=1000`).

# Data Flow
| Phase | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| **Configuration** | `args[0]` – path to properties file | Load DB connection parameters, table names, DB name | In‑memory configuration map |
| **Logging** | `args[2]` – log file path | Initialise `java.util.logging.FileHandler` with `SimpleFormatter` | Log file written to |
| **Connection** | `HIVE_HOST`, `HIVE_JDBC_PORT` | `hive_jdbc_connection.getJDBCConnection` | Open JDBC `Connection` to Hive |
| **Partition Management** | `args[3]` – current day, `args[4]` – previous month, `args[1]` – target partition date | Conditional `ALTER TABLE … DROP IF EXISTS PARTITION` statements | Stale partitions removed |
| **Data Load** | `usage_overage_charge_temp_tblname` (staging table) | `INSERT INTO … SELECT … FROM <temp_table>` | Rows inserted into `usage_overage_charge_tblname` partition `partition_date = args[1]` |
| **Cleanup** | `Connection`, `Statement`, `FileHandler` | `closeAll` | Resources released |

External services:
- Hive metastore accessed via JDBC.
- No message queues or external APIs.

# Integrations
- **Preceding step**: `usage_overage_charge_temp` utility populates the temporary staging table referenced here.
- **Subsequent step**: Downstream billing view aggregations (e.g., `customer_invoice_summary_estimated_temp`) read from `usage_overage_charge_tblname`.
- **Shared utilities**: `com.tcl.hive.jdbc.hive_jdbc_connection` provides connection handling and resource cleanup used across MOVE‑Mediation Java jobs.

# Operational Risks
- **Partition drop logic** – Drops the previous month partition only when `current_day` ≤ 05; if the job runs later, stale data may persist. *Mitigation*: Enforce schedule or add explicit cleanup job.
- **Hard‑coded Hive session settings** – May conflict with cluster‑wide defaults. *Mitigation*: Externalise to properties.
- **No retry on transient JDBC failures** – Job aborts on first exception. *Mitigation*: Implement retry/back‑off wrapper.
- **Logging to a single file without rotation** – Potential disk‑space exhaustion. *Mitigation*: Use log rotation or size‑based rollover.

# Usage
```bash
java -cp <classpath> com.tcl.mnaas.billingviews.usage_overage_charge \
    /path/to/usage_overage_charge.properties \
    2024-09-01 \
    /var/log/mnaas/usage_overage_charge.log \
    03 \
    2024-08
```
- `args[0]` – properties file containing DB connection and table names.  
- `args[1]` – target partition date (e.g., `2024-09-01`).  
- `args[2]` – absolute path to the log file.  
- `args[3]` – current day of month (`01`‑`31`).  
- `args[4]` – previous month identifier used for early‑month cleanup (e.g., `2024-08`).

# Configuration
Properties file keys (sample):
```properties
dbname=move_mediation
HIVE_JDBC_PORT=10000
HIVE_HOST=hive-prod.example.com
IMPALAD_JDBC_PORT=21050
IMPALAD_HOST=impala-prod.example.com
usage_overage_charge_tblname=usage_overage_charge
usage_overage_charge_temp_tblname=usage_overage_charge_temp
```
Only Hive connection parameters and table names are consumed; Impala settings are present but unused in this class.

# Improvements
1. **Externalise Hive session parameters** – Move `hiveDynamicPartitionMode` and `hiveDynamicPartitionPerNode` to the properties file to allow cluster‑specific tuning without code changes.  
2. **Add idempotent partition handling** – Before dropping a partition, verify its existence and optionally archive it, reducing risk of accidental data loss during re‑runs.