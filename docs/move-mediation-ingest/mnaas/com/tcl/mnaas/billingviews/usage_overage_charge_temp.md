**# Summary**  
`usage_overage_charge_temp.java` is a batch‑style Java driver that populates a temporary Hive/Impala table (`usage_overage_charge_temp`) with over‑age charge calculations for the previous month. It reads a properties file for connection details and table names, truncates the target table, then executes two very large Hive/Impala INSERT‑SELECT statements (one for the “current‑day‑1‑5” path and a second unconditional one). The statements join usage aggregates, customer‑overage flags, pricing slabs, and currency conversion tables to derive raw usage, over‑age usage, and monetary charges in both local currency and USD.

**# Key Components**  

- **Class `usage_overage_charge_temp`** – entry point (`main`) that orchestrates the load.  
- **Static Logger & FileHandler** – writes detailed execution logs to a file supplied on the command line.  
- **JDBC Connection Helpers (`hive_jdbc_connection`)** – provides Impala (or Hive) JDBC connections and clean‑up utilities.  
- **`main(String[] args)`** – parses arguments, loads properties, builds the JDBC connection, truncates the target table, decides which query to run, and executes the INSERT‑SELECT statements.  
- **`catchException(Exception, Connection, Statement)`** – centralised error handling that logs, closes resources and exits.  
- **`closeAll(Connection, Statement)`** – delegates resource shutdown to the JDBC helper.  

**# Data Flow**  

| Step | Input | Processing | Output / Side‑Effect |
|------|-------|-------------|----------------------|
| 1 | Command‑line: `<propertiesFile> <dayOfMonth> <logFile>` | `Properties.load()` reads DB connection strings, DB name, target table names. | In‑memory config values. |
| 2 | `dayOfMonth` (e.g., “01”) | Conditional check – if day ∈ {01‑05} the first large query (`query`) is executed; otherwise only `query1`. | Determines which INSERT‑SELECT runs. |
| 3 | JDBC connection to Impala (or Hive) | `stmt.execute("TRUNCATE TABLE …")` clears the temp table. | Temp table emptied. |
| 4 | Two massive INSERT‑SELECT strings (`query` & `query1`) | Each statement performs: <br>• Aggregation of raw CDR/traffic data (`traffic_aggr_adhoc`, `traffic_details_raw_daily_with_no_dups`). <br>• Join with `customers_with_overage_flag_table` to identify over‑age flags. <br>• Join with `bl_customer_airtime_usage_rates` (pricing slabs) and `gbs_curr_avgconv_rate` (FX). <br>• Compute over‑age usage and monetary charges (local & USD). | Rows inserted into `<dbname>.<usage_overage_charge_temp_tblname>`. |
| 5 | Logging | All major actions (property load, truncate, query strings, exceptions) are written to the supplied log file. | Audit trail for operations. |
| 6 | Resource cleanup | `closeAll` closes the JDBC `Statement`, `Connection`, and `FileHandler`. | No dangling resources. |

**External Services / Systems**  

- **Impala/Hive cluster** – executes the massive SQL statements.  
- **File system** – reads the properties file; writes the log file.  
- **No message queues or REST services** are used.

**# Integrations**  

- **`hive_jdbc_connection`** – shared library used across the move‑mediation codebase for establishing and tearing down JDBC connections to Hive/Impala.  
- **Other ETL jobs** – this script is typically invoked after the daily traffic aggregation jobs (`traffic_aggr_adhoc`, `traffic_details_raw_daily_with_no_dups`) have completed, because it depends on those tables being populated.  
- **Downstream billing processes** – the temporary table populated here is later consumed by billing view refreshes or charge‑generation jobs that read from `usage_overage_charge_temp`.  

**# Operational Risks**  

1. **SQL Statement Size / Parsing Failure** – the INSERT‑SELECT strings exceed typical line‑length limits; any minor syntax change can cause the whole job to fail.  
   *Mitigation*: Store the SQL in external `.hql` files and load them at runtime; validate with a dry‑run.  

2. **Resource Exhaustion on Impala** – the queries perform many joins, window functions, and full‑month scans; they can saturate cluster memory or cause long runtimes.  
   *Mitigation*: Ensure appropriate Impala memory limits, monitor query duration, and consider breaking the logic into staged intermediate tables.  

3. **Hard‑coded Date Logic** – uses `add_months(adddate(now(),0),-1)` and day‑of‑month checks; timezone drift or calendar changes could mis‑align the month being processed.  
   *Mitigation*: Pass the target month as an explicit parameter; use UTC consistently.  

4. **No Transactional Guarantees** – `TRUNCATE` followed by INSERT is not atomic; a failure after truncate leaves the temp table empty.  
   *Mitigation*: Load into a staging table and swap names on success, or use `INSERT OVERWRITE`.  

5. **Silent Failure on Partial Data** – if any of the source tables are missing partitions, the query may return zero rows without raising an error.  
   *Mitigation*: Add pre‑flight checks for expected partition existence and row counts.  

**# Usage**  

```bash
# Compile (once)
javac -cp $(hadoop classpath):./lib/* com/tcl/mnaas/billingviews/usage_overage_charge_temp.java

# Run
java -cp .:$(hadoop classpath):./lib/* \
     com.tcl.mnaas.billingviews.usage_overage_charge_temp \
     /path/to/usage_overage.properties 01 /var/log/usage_overage.log
```

- **Debugging**:  
  - Increase logger level (`logger.setLevel(Level.FINE)`) before compilation.  
  - Run the generated SQL statements directly in Impala/Beeline to isolate syntax errors.  
  - Verify that the properties file contains correct `HIVE_HOST`, `IMPALAD_HOST`, `dbname`, and table names.  

**# Configuration**  

Properties file (example keys used in the code):

| Property | Meaning |
|----------|---------|
| `dbname` | Hive/Impala database containing the target and source tables. |
| `HIVE_JDBC_PORT` / `HIVE_HOST` | (optional) Hive connection details – currently unused. |
| `IMPALAD_JDBC_PORT` / `IMPALAD_HOST` | Impala connection host/port. |
| `usage_overage_charge_temp_tblname` | Name of the temporary table