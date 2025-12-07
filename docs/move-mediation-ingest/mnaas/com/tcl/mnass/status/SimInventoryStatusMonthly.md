# Summary
`SimInventoryStatusMonthly` is a command‑line Java utility that loads monthly SIM inventory status data from an intermediate Hive table into a destination Hive table. It reads a Hive JDBC properties file, a partition‑date list file, and a log file path, then for each month it drops the existing partition, inserts SNG SIMs, non‑active HOL SIMs, and active HOL SIMs (with VIN enrichment) using prepared Hive INSERT‑SELECT statements. Temporary VIN mapping data is refreshed before processing. All JDBC resources and the log handler are closed on completion or error.

# Key Components
- **class `SimInventoryStatusMonthly`** – entry point; orchestrates configuration loading, logging, and execution flow.  
- **`main(String[] args)`** – parses arguments (`propertiesFile`, `processName`, `partitionsFile`, `logFile`), sets up logger, validates process name, reads partition months, invokes `updateSimInventoryMonthlyTable`.  
- **`updateSimInventoryMonthlyTable(List<String> partitionDates)`** – core ETL logic: prepares Hive statements, truncates/refreshes VIN temp table, iterates over each month, computes start/end dates, executes:
  - `alter table … drop if exists partition`
  - `insert … SNG SIMs`
  - `insert … non‑active HOL SIMs`
  - `insert … active HOL SIMs` (dynamic query built with month literal)
- **`getStackTrace(Exception e)`** – utility to convert exception stack trace to string for logging.  

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| Config load | Hive JDBC properties file (path from `args[0]`) | Reads host, port, DB name, source/destination table names | Populates static fields |
| Partition list | Text file (path from `args[2]`) | Reads each line as `yyyy-MM` month string | `List<String> partitionDates` |
| VIN temp refresh | Hive tables `mnaas.kyc_inter_raw_daily` | Truncate `sim_vin_mapping_temp`; insert latest VIN per ICCID | Populated temp table |
| Monthly load (per partition) | Source Hive table (`sourceTable`) | Executes three INSERT‑SELECT statements with joins to `active_sim_list` and `sim_vin_mapping_temp` | Inserts rows into destination table partition `partition_month = <yyyy-MM>` |
| Logging | Log file path (`args[3]`) | Java `java.util.logging` writes INFO/ALL level messages | Persistent execution log |

External services:
- Hive server accessed via JDBC (`hive_jdbc_connection.getJDBCConnection`).
- No message queues or REST services.

# Integrations
- **Hive JDBC connection utility** (`com.tcl.hive.jdbc.hive_jdbc_connection`) for obtaining and closing connections/statements.  
- **Other Hive tables** referenced:  
  - Source: `dbName.sourceTable` (move_sim_inventory_status)  
  - Destination: `dbName.destTable` (sim_inventory_status_monthly)  
  - Lookup: `mnaas.active_sim_list`, `mnaas.sim_vin_mapping_temp`, `mnaas.kyc_inter_raw_daily`  
- **Related batch jobs** (not in this file) that generate the intermediate source table and the partition list file.

# Operational Risks
- **SQL injection risk** – active query concatenates month literal directly; mitigated by strict month format validation (`yyyy-MM`).  
- **Partition date parsing errors** – `ParseException` aborts whole run; ensure input file conforms to `yyyy-MM`.  
- **Resource leaks on partial failure** – explicit close calls in catch block; however, if an exception occurs before statements are instantiated, null checks may be needed.  
- **Hard‑coded customer exclusions** (`cust_num not in (41218, 37226, 41648)`) – may become stale; maintain centrally.  
- **Dynamic partition mode only set to nonstrict** – may cause unexpected large partitions; monitor Hive execution plans.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.status.SimInventoryStatusMonthly \
    /path/to/hive.properties \
    sim_inventory_status_monthly \
    /path/to/partition_months.txt \
    /var/log/sim_inventory_monthly.log
```
- `args[0]` – Hive JDBC properties file.  
- `args[1]` – Process name (`sim_inventory_status_monthly`).  
- `args[2]` – File containing one `yyyy-MM` per line.  
- `args[3]` – Log file (appended).  

To debug, set logger level to `FINE` or add `System.out.println` statements; ensure Hive JDBC driver is on classpath.

# Configuration
- **Hive JDBC properties file** (keys used):
  - `HIVE_HOST`
  - `HIVE_JDBC_PORT`
  - `dbname`
  - `Move_siminventory_status_tblname`
  - `sim_inventory_status_monthly_tblname`
- No environment variables; all configuration is file‑based.  
- Log file path supplied as argument; rotates only by external log management.

# Improvements
1. **Parameterize customer exclusion list** – move hard‑coded IDs to properties file or database table to avoid code changes.  
2. **Refactor active SIM query construction** – use a prepared statement with a placeholder for the month literal instead of string concatenation, eliminating potential SQL injection and simplifying maintenance.