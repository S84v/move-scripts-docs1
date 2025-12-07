**# Summary**  
`billable_sub_with_charge_and_limit_temp_loading` is a standalone Java utility that populates a temporary Hive/Impala table (`billable_sub_with_charge_and_limit_temp_tblname`) with billable‑subscriber charge and limit data for a given processing day. It truncates the target table, then executes a large INSERT‑SELECT statement (or a series of them) that aggregates SIM activity, pricing, discounts, and currency conversion information from multiple `mnaas` Hive tables. The job is intended to run daily as part of the telecom “move‑mediation‑ingest” pipeline.

---

**# Key Components**  

- **Class `billable_sub_with_charge_and_limit_temp_loading`** – entry point (`main`).  
- **`main(String[] args)`** – parses arguments, loads properties, configures logging, obtains an Impala JDBC connection, truncates the target table, builds and executes the INSERT‑SELECT query (or a subset based on the day of month).  
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, exits with error.  
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` utility to close JDBC objects and the file handler.  

External utility: `com.tcl.hive.jdbc.hive_jdbc_connection` (provides `getImpalaJDBCConnection` and cleanup methods).

---

**# Data Flow**  

| Phase | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| **Configuration** | `args[0]` – path to a Java `.properties` file | `Properties.load` → values: `dbname`, Hive/Impala host/port, target table name | In‑memory config map |
| **Runtime Parameter** | `args[1]` – `current_day` (two‑digit day of month) | Determines whether the “first‑month” query (`query1`) is executed (days 01‑05) | Conditional query execution |
| **Logging** | `args[2]` – log file path | `FileHandler` + `SimpleFormatter` attached to `logger` | Log file written |
| **Database** | Impala/Hive cluster (host/port from properties) | JDBC connection → `Statement` → `TRUNCATE TABLE` + large `INSERT INTO … SELECT …` | Target temp table populated with aggregated rows |
| **External Tables (Hive)** | `mnaas.*` tables (e.g., `move_sim_inventory_status_monthly`, `bl_customer_pricing_master`, `gbs_curr_avgconv_rate`, etc.) | Complex SQL joins, CASE logic, currency conversion, prorating | Rows inserted into `billable_sub_with_charge_and_limit_temp_tblname` |

No message queues or file I/O beyond logging.

---

**# Integrations**  

- **Hive/Impala** – accessed via JDBC; all source tables reside in the `mnaas` database.  
- **`hive_jdbc_connection`** – shared library for connection handling; also used by other ingestion jobs in the move‑mediation suite.  
- **Properties file** – common across the move‑mediation pipeline; defines DB names, hosts, ports, and target table identifiers.  
- **Downstream** – the populated temporary table is later consumed by subsequent billing or reporting jobs (not shown in this file but referenced by naming convention).

---

**# Operational Risks**  

1. **SQL Complexity / Maintenance** – a single monolithic query is hard to read, test, and modify.  
   *Mitigation*: version‑control the query, add unit‑testable view definitions, or break into modular CTEs.  
2. **Resource Exhaustion** – large INSERT may cause Impala memory spikes or long runtimes, especially on first‑month runs.  
   *Mitigation*: enable dynamic partition mode, monitor query duration, consider incremental loads.  
3. **Hard‑coded Date Logic** – only days 01‑05 trigger `query1`; future calendar changes require code edits.  
   *Mitigation*: externalize the day‑range rule to the properties file.  
4. **No Retry / Idempotency** – on failure the job exits; partial truncation may leave the target table empty.  
   *Mitigation*: wrap truncate/insert in a transaction (if supported) or implement a “drop‑and‑recreate” fallback.  

---

**# Usage**  

```bash
# Prepare properties file (e.g., /opt/mnaas/conf/billable_sub.properties)
# Run the job
java -cp move-mediation-ingest.jar \
     com.tcl.mnaas.billingviews.billable_sub_with_charge_and_limit_temp_loading \
     /opt/mnaas/conf/billable_sub.properties 05 /var/log/mnaas/billable_sub.log
```

*Debugging*:  
- Set `logger.setLevel(Level.FINE)` in the source or modify the properties file to point to a test Impala instance.  
- Run with a reduced `current_day` (e.g., `01`) to trigger the first‑month query only.  

---

**# Configuration**  

| Property (in the supplied `.properties` file) | Description |
|----------------------------------------------|-------------|
| `dbname` | Hive/Impala database containing source and target tables. |
| `HIVE_JDBC_PORT` / `HIVE_HOST` | (Unused in current code; retained for compatibility). |
| `IMPALAD_JDBC_PORT` / `IMPALAD_HOST` | Host and port for the Impala daemon. |
| `billable_sub_with_charge_and_limit_temp_tblname` | Target temporary table to be truncated and loaded. |
| `traffic_details_daily_tblname` | Currently unused; may be referenced by other jobs. |

No environment variables are read directly; all runtime configuration is passed via the properties file.

---

**# Improvements**  

1. **Refactor SQL** – extract the massive INSERT‑SELECT into a stored Hive view or series of CTEs; store the query text in an external `.sql` file for easier editing and versioning.  
2. **Add Parameter Validation & Health Checks** – verify that required properties exist, that the target table is reachable, and that the Impala connection is alive before truncating; return a non‑zero exit code with a clear error message instead of `System.exit(1)` buried in a catch block.  