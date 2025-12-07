# Summary
`cdr_buid_search` is a command‑line Java utility that refreshes a Hive table containing Call Detail Records (CDR) indexed by Business Unit ID (BUID). For each date listed in a supplied partition‑date file, it drops the existing Hive partition for that date and inserts records from a temporary staging table. Execution is logged to a user‑provided log file and runs via a Hive JDBC connection.

# Key Components
- **class `cdr_buid_search`** – Main entry point; orchestrates configuration loading, logging, Hive connection, and ETL loop.  
- **`main(String[] args)`** – Parses arguments, reads properties, loads partition dates, establishes Hive JDBC connection, executes drop‑and‑insert statements per date, handles exceptions, and ensures resource cleanup.  
- **`catchException(Exception, Connection, Statement)`** – Logs the exception, closes resources, and exits with status 1.  
- **`closeAll(Connection, Statement)`** – Delegates to `hive_jdbc_connection` utility to close `Statement`, `Connection`, and `FileHandler`.  

# Data Flow
| Stage | Source | Destination | Operation |
|-------|--------|-------------|-----------|
| Config Load | Properties file (path = `args[0]`) | In‑memory variables (`dbname`, Hive host/port, table names) | `Properties.load` |
| Partition List | Text file (path = `args[1]`) | `List<String> partitionDates` | BufferedReader line‑by‑line |
| Hive Drop | Hive table `dbname.cdr_buid_seacrh_tblname` partition `calldate=event_date` | – | `ALTER TABLE … DROP IF EXISTS PARTITION` |
| Hive Insert | Hive staging table `dbname.cdr_buid_seacrh_temp_tblname` filtered by `calldate=event_date` | Hive target table `dbname.cdr_buid_seacrh_tblname` partition `(businessunitid, calldate)` | `INSERT INTO … SELECT … FROM … WHERE calldate='event_date'` |
| Logging | – | Log file (path = `args[2]`) | Java `Logger` with `FileHandler` |
| Side Effects | Hive metastore & data files | Updated partitions for each processed date | Table metadata & HDFS data modifications |

# Integrations
- **`hive_jdbc_connection`** – Custom wrapper providing `getJDBCConnection`, `closeStatement`, `closeConnection`, and `closeFileHandler`.  
- **Hive/Impala** – Executes HiveQL via JDBC; can be switched to Impala by uncommenting the Impala connection line.  
- **External scripts** – Typically invoked by a scheduler (e.g., Oozie, Airflow) that supplies the properties file, partition list, and log path.

# Operational Risks
- **Missing or malformed partition file** → job aborts; ensure file existence and correct date format.  
- **Hive partition drop without backup** → data loss if insert fails; consider staging inserts before drop or using `INSERT OVERWRITE`.  
- **Hard‑coded dynamic partition settings** may not match cluster configuration; verify `hive.exec.max.dynamic.partitions.pernode` suitability.  
- **Uncontrolled resource leakage** if `catchException` fails before `closeAll`; already mitigated by finally block but monitor for JVM OOM.  

# Usage
```bash
java -cp <classpath> com.tcl.mnass.tableloading.cdr_buid_search \
    /path/to/config.properties \
    /path/to/partition_dates.txt \
    /var/log/cdr_buid_search.log
```
- `config.properties` must contain keys: `dbname`, `HIVE_JDBC_PORT`, `HIVE_HOST`, `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`, `cdr_buid_seacrh_tblname`, `cdr_buid_seacrh_temp_tblname`.  
- `partition_dates.txt` – one `yyyy-MM-dd` value per line.  

# Configuration
- **Properties file** (first CLI argument) – defines Hive/Impala connection parameters, database name, and target/temp table names.  
- **Environment** – Java 8+, HiveServer2 reachable at `HIVE_HOST:HIVE_JDBC_PORT`.  
- **Log file** – third CLI argument; appended on each run.  

# Improvements
1. **Idempotent load** – Replace drop‑then‑insert with `INSERT OVERWRITE` to avoid a window of missing data if the insert fails.  
2. **Parameter validation & retry logic** – Add pre‑flight checks for file existence, date format validation, and configurable retry on transient SQL exceptions.