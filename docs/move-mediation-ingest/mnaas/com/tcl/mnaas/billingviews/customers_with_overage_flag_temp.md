# Summary
`customers_with_overage_flag_temp` is a standalone Java utility that populates a temporary Hive/Impala table with customer usage aggregates and over‑age flags. It reads runtime parameters and a properties file, connects to Impala via JDBC, truncates the target table, and executes one or more large INSERT‑SELECT statements that join traffic aggregation data with billing limits. The result is a refreshed “customers with overage flag” view used downstream in the telecom billing pipeline.

# Key Components
- **`public static void main(String[] args)`** – orchestrates configuration loading, logging setup, JDBC connection, table truncation, conditional query execution, and final cleanup.  
- **`public static void catchException(Exception e, Connection c, Statement s)`** – centralised error handling: logs the exception, closes resources, and exits with status 1.  
- **`public static void closeAll(Connection c, Statement s)`** – delegates to `hive_jdbc_connection` helpers to close the `Statement`, `Connection`, and `FileHandler`.  
- **Static fields** – logger, file handler, JDBC host/port constants, Hive dynamic‑partition settings, and table‑name placeholders populated from the properties file.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **Argument parsing** | `args[0]` – properties file path  <br> `args[1]` – partitions file path (used as a literal string in the SQL) <br> `args[2]` – log file path <br> `args[3]` – `current_day` (DD) <br> `args[4]` – `curyr_prevmon` (YYYYMM) | Load `Properties`, initialise logger, assign variables. | Populated in‑memory configuration. |
| **JDBC connection** | Host/port from properties (`IMPALAD_HOST`, `IMPALAD_JDBC_PORT`) | `hive_jdbc_connection.getImpalaJDBCConnection` returns a live `java.sql.Connection`. | Open Impala connection. |
| **Table truncation** | Target DB & table (`dbname`.`customers_with_overage_flag_temp_tblname`) | Execute `TRUNCATE TABLE …`. | Table emptied. |
| **Query selection** | `current_day` value | If day ∈ {01‑05} execute `query1` (previous‑month data). Always execute `query` (current‑month data). | Determines which INSERT‑SELECT statements run. |
| **SQL execution** | Large INSERT‑SELECT strings (`query1`, `query`) | `Statement.execute(sql)` runs the Hive/Impala query. | Rows inserted into the temporary table; over‑age flags calculated. |
| **Cleanup** | Open `Connection`, `Statement`, `FileHandler` | `closeAll` → `hive_jdbc_connection` helpers. | Resources released; log file closed. |

External services / DBs:
- **Impala/Hive cluster** (`mnaas` schema, tables `traffic_aggr_adhoc`, `bl_cust_bu_co_cogrp_mapping`, `billable_sub_with_charge_and_limit_table`, `bl_customer_pricing_master`).
- **Logging** – writes to the file supplied in `args[2]`.

# Integrations
- **`hive_jdbc_connection`** – custom utility class handling JDBC driver loading, connection lifecycle, and resource cleanup.  
- **Orchestration layer** – typically invoked by a scheduler (e.g., Oozie, Airflow, cron) that supplies the five command‑line arguments.  
- **Downstream consumers** – other billing jobs read the populated `customers_with_overage_flag_temp` table to generate invoices, alerts, or analytics. No direct message‑queue interaction.

# Operational Risks
1. **SQL string concatenation** – Direct insertion of file‑path and date strings can cause malformed queries if inputs contain unexpected characters. *Mitigation*: validate/sanitize arguments; consider prepared statements or external SQL templates.  
2. **Hard‑coded date logic** – Only the first five days trigger the “previous‑month” load; any change in business calendar requires code change. *Mitigation*: externalize the rule to a config flag.  
3. **Large monolithic queries** – May exceed Impala’s query size limits or cause long execution times, leading to timeouts. *Mitigation*: break into modular sub‑queries or use temporary staging tables.  
4. **No retry / idempotency** – Failure after truncation leaves the target table empty. *Mitigation*: implement transactional semantics or a “drop‑and‑recreate” pattern with backup.  
5. **Resource leakage on abnormal termination** – `System.exit(1)` bypasses JVM shutdown hooks. *Mitigation*: rely on try‑with‑resources or ensure finally block always runs.

# Usage
```bash
# Compile (assuming required jars are in lib/)
javac -cp "lib/*" com/tcl/mnaas/billingviews/customers_with_overage_flag_temp.java

# Run
java -cp ".:lib/*" com.tcl.mnaas.billingviews.customers_with_overage