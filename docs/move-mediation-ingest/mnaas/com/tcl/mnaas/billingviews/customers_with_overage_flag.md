# Summary
`customers_with_overage_flag` is a Java utility that refreshes the Hive table `customers_with_overage_flag` for a given processing date. It drops stale partitions (including a fallback for early‑month dates), then inserts the latest over‑age flag data from a temporary staging table. The job is invoked from a shell/cron step in the MOVE‑Mediation Ingest pipeline.

# Key Components
- **class `customers_with_overage_flag`** – entry point and orchestrator.  
- **`main(String[] args)`** – parses arguments, loads properties, configures logging, establishes Hive JDBC connection, executes partition drop and insert statements.  
- **`catchException(Exception, Connection, Statement)`** – central error handling; logs, cleans up, exits with status 1.  
- **`closeAll(Connection, Statement)`** – closes JDBC resources and the file handler via `hive_jdbc_connection` helper.  
- **Static members** – logger, file handler, Hive/Impala connection parameters, Hive session settings, and numeric precision constant.

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| 1 | Property file (path supplied as `args[0]`) | Load DB name, Hive/Impala hosts/ports, target table names. | In‑memory configuration map. |
| 2 | Partition file path (`args[1]`) | Treated as partition date string for drop/insert. | Used in Hive `ALTER TABLE … DROP IF EXISTS PARTITION`. |
| 3 | Log file path (`args[2]`) | `FileHandler` attached to `java.util.logging.Logger`. | Persistent execution log. |
| 4 | Current day (`args[3]`) & previous month date (`args[4]`) | Conditional early‑month partition drop. | Additional `DROP PARTITION` if day ≤ 05. |
| 5 | Hive JDBC connection (via `hive_jdbc_connection.getJDBCConnection`) | Execute Hive session settings, drop partitions, run `INSERT INTO … SELECT … FROM <temp_table>`. | Updated `customers_with_overage_flag` partition for the supplied date. |
| 6 | No explicit output object | All work performed in Hive; job returns exit code 0 on success, 1 on failure. | Data persisted in Hive metastore / HDFS. |

# Integrations
- **`hive_jdbc_connection`** – utility class providing JDBC connection management for Hive (and optionally Impala).  
- **Hive Metastore / HDFS** – target tables reside in the configured Hive database (`dbname`).  
- **External property file** – shared configuration across MOVE‑Mediation ingest jobs (e.g., table names, host/port).  
- **Logging infrastructure** – writes to a file defined by the caller; consumed by monitoring/alerting pipelines.  
- **Shell/cron orchestrator** – typical upstream script passes the required arguments and triggers the Java class.

# Operational Risks
- **Hard‑coded Hive session settings** – changes to Hive dynamic partition limits require code modification. *Mitigation*: externalize settings to properties.  
- **No validation of input arguments** – malformed dates or missing files cause uncaught exceptions before logging. *Mitigation*: add pre‑flight checks.  
- **Single‑threaded JDBC execution** – large inserts may exceed default Hive query timeout. *Mitigation*: configure timeout via Hive settings or split insert into batches.  
- **Immediate `System.exit(1)` on any exception** – may abort downstream jobs. *Mitigation*: return error codes and allow orchestrator to decide.  
- **Potential resource leak if `hive_jdbc_connection.close*` fails** – could leave open connections. *Mitigation*: use try‑with‑resources or ensure close methods swallow exceptions.

# Usage
```bash
# Compile (if not already packaged)
javac -cp <hive-jdbc-jar>:<project-jar> com/tcl/mnaas/billingviews/customers_with_overage_flag.java

# Run
java -cp <all‑jars> com.tcl.mnaas.billingviews.customers_with_overage_flag \
    /path/to/mnaas.properties \
    2024-09-01 \
    /var/log/mnaas/customers_overage.log \
    01 \
    2024-08-01
```
- Argument 0: absolute path to the properties file.  
- Argument 1: partition date string (used for drop/insert).  
- Argument 2: log file path (appended).  
- Argument 3: current day of month (`01`‑`31`).  
- Argument 4: previous month same‑day string (used for early‑month fallback).

# Configuration
Properties file (example keys):
```properties
dbname=move_mediation
HIVE_JDBC_PORT=10000
HIVE_HOST=hive-prod.example.com
IMPALAD_JDBC_PORT=21050
IMPALAD_HOST=impala-prod.example.com
customers_with_overage_flag_tblname=customers_with_overage_flag
customers_with_overage_flag_temp_tblname=customers_with_overage_flag_temp
```
No environment variables are read directly; all runtime parameters are supplied via command‑line arguments.

# Improvements
1. **Externalize Hive session commands** – move `hiveDynamicPartitionMode` and related settings to the properties file to avoid recompilation for configuration changes.  
2. **Refactor to use try‑with‑resources** – replace manual close calls with Java 7+ resource management to guarantee deterministic cleanup and simplify error handling.