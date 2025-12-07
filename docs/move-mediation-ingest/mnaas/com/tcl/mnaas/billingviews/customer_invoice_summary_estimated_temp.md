# Summary
The `customer_invoice_summary_estimated_temp` Java program truncates a target Hive/Impala table and populates it with aggregated billing data for the current and previous month. It builds two large INSERT‑SELECT statements that join billable subscription, usage overage, and service‑based charge tables, enriches with currency conversion and commercial‑offer mappings, and writes the results into the temporary summary table used downstream in the telecom move‑mediation pipeline.

# Key Components
- **class `customer_invoice_summary_estimated_temp`** – entry point; orchestrates DB connection, logging, and query execution.  
- **`main(String[] args)`** – parses arguments, loads properties, configures logger, obtains Impala JDBC connection, truncates target table, builds and executes two INSERT queries (`query` for previous month, `query1` for current month).  
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, exits with error.  
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` utility to close statement, connection, and file handler.  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|-------------|----------------------|
| **Configuration** | Property file path (arg 0) | Loads DB name, host/port, target table name | In‑memory config variables |
| **Logging** | Log file path (arg 2) | `FileHandler` + `SimpleFormatter` attached to `logger` | Log entries written to file |
| **DB Connection** | Hive/Impala host & port (args) | `hive_jdbc_connection.getImpalaJDBCConnection` | Open JDBC `Connection` |
| **Truncate** | Target table name | `TRUNCATE TABLE db.tbl` | Table emptied |
| **Query 1** (previous month) | Partition date = previous month (`add_months(...,-1)`) | Large `INSERT … SELECT` aggregating billable, usage, service, and SEBS charge data; joins currency conversion and commercial‑offer mapping | Rows inserted into temp table for previous month |
| **Query 2** (current month) | Partition date = current month (`add_months(...,0)`) | Same aggregation logic scoped to current month | Additional rows inserted for current month |
| **Cleanup** | JDBC resources | `closeAll` | Connection, statement, and file handler closed |

External services:
- Impala/Hive cluster (via JDBC)
- `hive_jdbc_connection` utility library (connection & cleanup)

# Integrations
- **Upstream**: Data populated in source tables (`billable_sub_with_charge_and_limit_table`, `usage_overage_charge_table`, `service_based_charges_table`, `sebs_chn_dtl_invce_reg`, `gbs_curr_avgconv_rate`, `bl_cust_bu_co_cogrp_mapping`) by earlier ETL jobs in the move‑mediation workflow.  
- **Downstream**: The populated temporary table (`customer_invoice_summary_estimated_temp_tblname`) is consumed by subsequent billing or reporting jobs that generate final customer invoices.  
- **Shared utilities**: `com.tcl.hive.jdbc.hive_jdbc_connection` provides common connection handling across the suite.

# Operational Risks
- **SQL string concatenation** – risk of injection or malformed queries if property values contain unexpected characters. *Mitigation*: validate/escape config values; consider prepared statements for static parts.  
- **Long‑running query** – massive INSERT may exceed Impala timeout or cause resource contention. *Mitigation*: monitor query duration; tune `hive.execution.engine` or enable query retries.  
- **Hard‑coded date logic** – uses `adddate(now(),0)` and string formatting; timezone changes could mis‑align partitions. *Mitigation*: enforce UTC and document expected schedule.  
- **Single point of failure** – program exits on any exception, potentially leaving target table partially populated. *Mitigation*: implement transactional staging (write to staging table then swap).  

# Usage
```bash
# Prepare property file (e.g., /opt/conf/billing.properties) with required keys:
#   dbname, HIVE_JDBC_PORT, HIVE_HOST, IMPALAD_JDBC_PORT, IMPALAD_HOST,
#   customer_invoice_summary_estimated_temp_tblname
java -cp move-mediation-ingest.jar com.tcl.mnaas.billingviews.customer_invoice_summary_estimated_temp \
    /opt/conf/billing.properties 01 /var/log/billing_summary.log
```
- `args[0]` – path to properties file  
- `args[1]` – day of month (used to decide if previous‑month logic runs)  
- `args[2]` – log file path  

For debugging, run with a JVM debugger attached or add `System.out.println` statements before query execution.

# Configuration
| Property | Description |
|----------|-------------|
| `dbname` | Hive/Impala database containing target table |
| `HIVE_JDBC_PORT` / `HIVE_HOST` | (unused in current code) |
| `IMPALAD_JDBC_PORT` / `IMPALAD_HOST` | Impala service endpoint |
| `customer_invoice_summary_estimated_temp_tblname` | Target temporary table name |
| *Implicit* | No environment variables; all configuration via the supplied properties file. |

# Improvements
1. **Refactor query construction** – extract the massive INSERT statements into external `.sql` templates and load them at runtime; use parameter binding for dates and table names to improve readability and maintainability.  
2. **Add retry & idempotency** – wrap each INSERT in a try‑catch with retry logic; write to a staging table and atomically replace the target to avoid partial loads on failure.