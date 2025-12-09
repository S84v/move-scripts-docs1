# Summary
The `siminventory_insertion` Java utility loads SIM inventory data from an intermediate Hive table into a target Hive table, partitioned by a supplied date value. It establishes a JDBC connection to Hive, configures dynamic partitioning, builds and executes an INSERT‑SELECT statement that transforms timestamps to UTC and applies simple data sanitisation, then logs progress and cleans up resources.

# Key Components
- **class `siminventory_insertion`** – entry point containing `main`, logging setup, and helper methods.
- **`main(String[] args)`** – parses arguments, loads properties, opens Hive JDBC connection, sets Hive session parameters, builds the INSERT query, executes it, and handles exceptions.
- **`catchException(Exception, Connection, Statement)`** – logs the exception, closes resources, and exits with status 1.
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` utilities to close statement, connection, and file handler.
- **Static fields** – configuration strings (`HIVE_HOST`, `HIVE_JDBC_PORT`, `db`, `src_tableName`, `dest_tableName`, etc.) and Hive session settings (`hiveDynamicPartitionMode`, `partitionsPernode`).

# Data Flow
1. **Inputs**  
   - `args[0]`: Path to a Java `.properties` file containing Hive connection details and table names.  
   - `args[1]`: Partition value (e.g., `2023-09`).  
   - `args[2]`: Path to a log file.  
2. **Processing**  
   - Loads properties → establishes Hive JDBC connection.  
   - Executes `SET hive.exec.dynamic.partition.mode=nonstrict` and `SET hive.exec.max.dynamic.partitions.pernode=1000`.  
   - Constructs an `INSERT INTO <db>.<dest_table> PARTITION(partition_date) SELECT … FROM <db>.<src_table>` query, applying:  
     - `coalesce` for numeric defaults,  
     - `if(... = 'null', '', …)` for string sanitisation,  
     - `from_utc_timestamp(date_format(...,'yyyy-MM-dd HH:mm:ss.SSS'),'UTC')` for timestamp conversion,  
     - static partition literal (`'partition'`).  
   - Executes the query, loading data into the target table partition.  
3. **Outputs / Side Effects**  
   - Populates the Hive table `<db>.<dest_table>` with transformed rows under the specified partition.  
   - Writes operational logs to the supplied log file.  

# Integrations
- **Hive JDBC driver** (`com.tcl.hive.jdbc.hive_jdbc_connection`) – provides connection, statement, and cleanup utilities.  
- **Hive Metastore** – target and source tables reside in the Hive warehouse (`mnaas.db`).  
- **External property file** – supplies runtime configuration (host, port, DB name, source/destination table names).  
- **Potential downstream consumers** – any analytics or reporting jobs that read from the partitioned `dest_table`.  

# Operational Risks
- **Schema drift** – SELECT list assumes a fixed column order in the source table; schema changes will cause query failures. *Mitigation*: validate source schema before execution or generate column list dynamically.  
- **Partition value misuse** – static partition literal from `args[1]` may be incorrect, leading to data landing in wrong partition. *Mitigation*: enforce date format validation.  
- **Resource exhaustion** – `SET hive.exec.max.dynamic.partitions.pernode=1000` may be insufficient for very large loads, causing job abort. *Mitigation*: monitor job logs and adjust the setting per cluster capacity.  
- **Uncaught runtime exceptions** – `System.exit(1)` terminates the JVM, potentially leaving dangling connections if `closeAll` fails. *Mitigation*: add finally block safeguards and use try‑with‑resources where possible.  

# Usage
```bash
# Prepare properties file (e.g., mnaas.properties) with keys:
# HIVE_HOST, HIVE_JDBC_PORT, dbname,
# Move_siminventory_status_inter_tblname, Move_siminventory_status_tblname

java -cp <classpath> com.tcl.mnass.tableloading.siminventory_insertion \
    /path/to/mnaas.properties \
    2023-09 \
    /var/log/siminventory_insertion.log
```
- Verify log file for `Successfully loaded` message.  
- To debug, set `logger.setLevel(Level.FINE)` or add `System.out.println` statements before query execution.

# Configuration
- **Properties file keys**  
  - `HIVE_HOST` – Hive server hostname.  
  - `HIVE_JDBC_PORT` – Hive Thrift port.  
  - `dbname` – Hive database name.  
  - `Move_siminventory_status_inter_tblname` – Source intermediate table.  
  - `Move_siminventory_status_tblname` – Destination table.  
- **Environment** – Java 8+, Hive JDBC driver on classpath, network access to Hive server.  

# Improvements
1. **Schema‑driven query generation** – Retrieve column metadata from the source table and build the SELECT list programmatically to protect against schema changes.  
2. **Enhanced error handling & observability** – Replace `System.exit` with proper exception propagation, add metrics (rows inserted, duration), and integrate with a monitoring system (e.g., Prometheus).