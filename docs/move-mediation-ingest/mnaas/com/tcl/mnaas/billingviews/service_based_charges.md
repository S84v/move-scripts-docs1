# Summary
`service_based_charges` is a Java utility executed in the MOVE‑Mediation ingest pipeline. It connects to Hive via JDBC, optionally drops a stale partition for the target table `service_based_charges_tblname`, then inserts aggregated service‑based charge data from a temporary staging table `service_based_charges_temp_tblname` into the target table partitioned by `partition_date`. The utility is invoked with a properties file, a partition‑date argument, a log file path, the current day of month, and the previous month identifier.

# Key Components
- **class `service_based_charges`** – entry point containing `main`, exception handling, and resource cleanup.  
- **`main(String[] args)`** – parses arguments, loads properties, configures logger, establishes Hive JDBC connection, executes dynamic partition settings, conditional partition drop, and final `INSERT … SELECT`.  
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, exits with status 1.  
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` helpers to close statement, connection, and file handler.  

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | `args[0]` – path to properties file | Load JDBC host/port, DB name, table names | In‑memory `Properties` |
| 2 | `args[1]` – partition date string (e.g., `2023-09-01`) | Used in `DROP PARTITION` statement | Hive partition removal |
| 3 | `args[2]` – log file path | Configure `java.util.logging.FileHandler` | Log file creation |
| 4 | `args[3]` – current day (`01`‑`05` etc.) | Determines whether previous‑month partition is dropped | Conditional `DROP PARTITION` |
| 5 | `args[4]` – previous month identifier (`YYYY-MM`) | Used in conditional `DROP PARTITION` | Conditional `DROP PARTITION` |
| 6 | Hive JDBC connection (via `hive_jdbc_connection.getJDBCConnection`) | Execute `SET` statements for dynamic partition mode | Session configuration |
| 7 | `service_based_charges_tblname` & `service_based_charges_temp_tblname` from properties | `INSERT INTO … SELECT` from temp to target table, partitioned by `partition_date` | Populated target Hive table |

External services: Hive metastore accessed through JDBC; no message queues or file I/O beyond logging.

# Integrations
- **`hive_jdbc_connection`** – utility class providing JDBC connection creation and resource cleanup; shared across MOVE‑Mediation utilities.  
- **Other billing view utilities** (`customers_with_overage_flag_temp`, `customer_invoice_summary_estimated`, etc.) write to the same Hive database and may depend on the populated `service_based_charges` table for downstream joins.  
- **Scheduling/orchestration** (e.g., Oozie, Airflow) invokes this Java program as part of a daily/weekly batch job, passing the required arguments.

# Operational Risks
- **Hard‑coded Hive settings** – changes to Hive dynamic partition limits require code modification. *Mitigation*: externalize settings to properties.  
- **Unvalidated arguments** – missing or malformed args cause `ArrayIndexOutOfBoundsException` before error handling. *Mitigation*: add argument count check and validation.  
- **Single‑threaded execution** – large `INSERT … SELECT` may lock target table, impacting concurrent jobs. *Mitigation*: schedule non‑overlapping windows or use transactional tables.  
- **No retry logic** – transient JDBC failures abort the job. *Mitigation*: implement retry with exponential back‑off.  

# Usage
```bash
java -cp move-mediation-ingest.jar \
     com.tcl.mnaas.billingviews.service_based_charges \
     /path/to/service_based_charges.properties \
     2023-09-01 \
     /var/log/mnaas/service_based_charges.log \
     03 \
     2023-08
```
- `03` (current_day) triggers drop of previous‑month partition (`2023-08`).  
- Adjust classpath to include Hive JDBC driver and `hive_jdbc_connection` library.

# Configuration
Properties file (example keys):
```
dbname=telecom_billing
HIVE_JDBC_PORT=10000
HIVE_HOST=hive-prod.example.com
IMPALAD_JDBC_PORT=21050
IMPALAD_HOST=impala-prod.example.com
service_based_charges_tblname=service_based_charges
service_based_charges_temp_tblname=service_based_charges_temp
```
Only Hive connection parameters and table names are read; other settings are hard‑coded.

# Improvements
1. **Externalize Hive session settings** (dynamic partition mode, max partitions) to the properties file to avoid recompilation for environment changes.  
2. **Add robust argument validation and usage help**; return a non‑zero exit code with a clear message when required arguments are missing or malformed.