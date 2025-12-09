# Summary
`traffic_raw_table_without_dups_inter_loading` is a Java utility that loads deduplicated raw traffic records from the source Hive table `mnaas.traffic_details_raw_daily` into the intermediate Hive table `traffic_details_inter_raw_daily_with_no_dups`. For each partition date supplied in a file, it refreshes a KYC snapshot, inserts the deduplicated rows using a parametrised INSERT‑SELECT query, and writes progress to a log file. The program runs via a Hive JDBC connection and exits with status 1 on any error.

# Key Components
- **class `traffic_raw_table_without_dups_inter_loading`**
  - `main(String[] args)`: orchestrates configuration loading, logging, JDBC connection, truncation of temp tables, per‑partition processing, and cleanup.
  - `catchException(Exception, Connection, Statement)`: logs the exception, closes resources, and terminates the JVM with exit code 1.
  - `closeAll(Connection, Statement)`: delegates to `hive_jdbc_connection` helpers to close statement, connection, and file handler.
- **`hive_jdbc_connection` (external utility)**
  - Provides `getJDBCConnection` for Hive, and resource‑cleanup methods.
- **`QueryReader` (external utility)**
  - Reads a YAML file (`query.yml`) and returns a parametrised SQL string for a given query key (e.g., `kyc_feed_mapping_snapshot`, `without_dups_inter_loading`).

# Data Flow
| Stage | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| Config load | `args[0]` – path to properties file | `Properties.load` | `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `traffic_details_inter_raw_daily_with_no_dups_tblname` |
| Partition list | `args[1]` – file with partition dates (one per line) | BufferedReader → `List<String> partitionDates` | List of dates to process |
| Logging | `args[2]` – log file path | `FileHandler` attached to `logger` | Log file with INFO/ERROR |
| Hive connection | `hiveHost`, `hiveJdbcPort` | `hive_jdbc_connection.getJDBCConnection` | `java.sql.Connection` |
| Temp table preparation | – | `TRUNCATE TABLE mnaas.sim_type_mapping_temp` then `INSERT INTO ... SELECT DISTINCT sim, sim_type FROM mnaas.move_sim_inventory_status WHERE partition_date = (SELECT MAX(partition_date) …)` | Populated temp mapping table |
| Target intermediate table truncation | `dbname`, `trafficDetailsInterRawDailyWithNoDupsTableName` | `TRUNCATE TABLE <db>.<table>` | Empty target table |
| Per‑partition loop | Each `partitionDate` | 1. Load KYC snapshot via `kyc_feed_mapping_snapshot` query (parameterised with partitionDate). 2. Load deduplicated traffic via `without_dups_inter_loading` query (parameterised with partitionDate). | Populated rows in `traffic_details_inter_raw_daily_with_no_dups` for the partition |
| Cleanup | – | `closeAll` → close JDBC resources and file handler | No open connections after exit |

External services:
- Hive metastore accessed via JDBC.
- YAML query repository at `/app/hadoop_users/MNAAS/MNAAS_Property_Files/query.yml`.

# Integrations
- **Hive/Impala tables**: reads from `mnaas.move_sim_inventory_status`, `mnaas.traffic_details_raw_daily`, `mnaas.kyc_mapping_snapshot`; writes to `mnaas.sim_type_mapping_temp` and `traffic_details_inter_raw_daily_with_no_dups`.
- **QueryReader**: shares the same `query.yml` used by other loading utilities (e.g., `traffic_raw_table_without_dups_inter_loading`, `table_statistics_*`).
- **Logging infrastructure**: integrates with the central log directory via the supplied log file path.
- **Configuration**: same property file format as other `tableloading` utilities (e.g., `table_statistics_customer_loading`).

# Operational Risks
- **Unbounded partition list**: Very large `partitionsFilenameWithPath` may cause OOM due to loading all dates into memory. *Mitigation*: stream dates or batch process.
- **Hard‑coded absolute paths** for `query.yml`. Deployment to a different environment may break. *Mitigation*: make path configurable.
- **No explicit transaction handling**; failures mid‑loop leave partially loaded partitions. *Mitigation*: wrap per‑partition inserts in a transaction or add idempotent cleanup.
- **Static truncation of target table** before loop; if the job crashes, data for earlier partitions is lost. *Mitigation*: truncate after successful load of all partitions or use staging tables.
- **Missing resource cleanup on `PreparedStatement`** (only `Statement` closed). *Mitigation*: close prepared statements explicitly.

# Usage
```bash
# Compile (assuming Maven/Gradle build includes dependencies)
javac -cp <classpath> com/tcl/mnass/tableloading/traffic_raw_table_without_dups_inter_loading.java

# Run
java -cp <classpath> com.tcl.mnass.tableloading.traffic_raw_table_without_dups_inter_loading \
    /path/to/mnaas.properties \
    /path/to/partition_dates.txt \
    /var/log/mnaas/traffic_raw_load.log
```
- `mnaas.properties` must contain keys: `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `traffic_details_inter_raw_daily_with_no_dups_tblname`.
- `partition_dates.txt` contains one `yyyy-MM-dd` value per line.

# Configuration
- **Properties file (arg 0)**
  - `dbname` – Hive database containing the target intermediate table.
  - `HIVE_JDBC_PORT` – Port for HiveServer2.
  - `HIVE_HOST` – Hostname for HiveServer2.
  - `traffic_details_inter_raw_daily_with_no_dups_tblname` – Target table name.
- **Query file**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/query.yml` – YAML with keys:
    - `kyc_feed_mapping_snapshot`
    - `without_dups_inter_loading`
- **Log file (arg 2)**
  - Path to a writable file; logger appends.

# Improvements
1. **Resource Management** – Use try‑with‑resources for `Connection`, `Statement`, and `PreparedStatement` to guarantee closure and reduce boilerplate.
2. **Parameterisation of Query File Path** – Add a command‑line argument or property to specify the location of `query.yml`, avoiding hard‑coded absolute paths and improving portability.