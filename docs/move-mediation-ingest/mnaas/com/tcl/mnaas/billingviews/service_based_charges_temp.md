# Summary
`service_based_charges_temp` is a Java utility executed in the MOVE‑Mediation ingest pipeline. It connects to Impala via JDBC, truncates the staging Hive/Impala table `service_based_charges_temp_tblname`, and populates it with aggregated service‑based charge records for the previous month (if the current day is 01‑05) and the current month. The data is inserted using two large `INSERT … SELECT` statements that join charge, pricing, and currency‑conversion tables.

# Key Components
- **class `service_based_charges_temp`** – entry point; contains `main`, exception handling, and resource cleanup.
- **`main(String[] args)`** – parses arguments, loads properties, configures logger, establishes JDBC connection, truncates target table, builds and executes two INSERT queries.
- **`catchException(Exception, Connection, Statement)`** – logs exception, closes resources, exits with status 1.
- **`closeAll(Connection, Statement)`** – delegates to `hive_jdbc_connection` utilities for statement, connection, and file‑handler cleanup.

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | 1. Properties file (path supplied as `args[0]`).<br>2. Current day string (`args[1]`).<br>3. Log file path (`args[2]`). |
| **Processing** | - Load Hive/Impala connection parameters from properties.<br>- Truncate `dbname.service_based_charges_temp_tblname`.<br>- If `current_day` ∈ {01‑05}, execute INSERT for previous month.<br>- Always execute INSERT for current month.<br>- Each INSERT selects from `mnaas.bl_customer_service_based_charges`, joins `mnaas.bl_customer_pricing_master` for currency, and joins `mnaas.gbs_curr_avgconv_rate` for USD conversion. |
| **Output** | Populated rows in `service_based_charges_temp_tblname` (partition‑less temporary table). |
| **Side Effects** | - Table truncation (data loss of previous temp contents).<br>- Writes to log file. |
| **External Services** | Impala/Hive cluster accessed via JDBC (`hive_jdbc_connection`). |

# Integrations
- **Downstream**: `service_based_charges` utility reads from this temporary table to load the final partitioned `service_based_charges` Hive table.
- **Upstream**: Relies on source tables `bl_customer_service_based_charges`, `bl_customer_pricing_master`, and `gbs_curr_avgconv_rate` in the `mnaas` database.
- **Shared Library**: `com.tcl.hive.jdbc.hive_jdbc_connection` provides connection handling and cleanup methods.

# Operational Risks
- **Data loss**: Truncation removes any previously staged data; failure after truncate but before successful insert leaves table empty. *Mitigation*: Add transactional staging or checkpoint before truncate.
- **Hard‑coded date logic**: Only inserts previous month for days 01‑05; if pipeline runs later, prior month data may be missing. *Mitigation*: Parameterize month range.
- **SQL injection risk**: Queries are built via string concatenation with property values; malformed properties could break SQL. *Mitigation*: Validate/escape property inputs.
- **Resource leakage**: If `hive_jdbc_connection` methods fail, connections may remain open. *Mitigation*: Ensure `finally` block always executes and consider try‑with‑resources.

# Usage
```bash
java -cp <classpath> com.tcl.mnaas.billingviews.service_based_charges_temp \
    /path/to/service_based_charges_temp.properties \
    $(date +%d) \
    /var/log/mnaas/service_based_charges_temp.log
```
- Replace `<classpath>` with compiled classes and required JARs (JDBC driver, logging, etc.).
- For debugging, set logger level to `FINE` or add `System.out.println` statements before each query execution.

# Configuration
Properties file keys (example):
```properties
dbname=your_hive_db
HIVE_JDBC_PORT=10000
HIVE_HOST=hive-host.example.com
IMPALAD_JDBC_PORT=21050
IMPALAD_HOST=impala-host.example.com
service_based_charges_temp_tblname=service_based_charges_temp
```
No environment variables are referenced directly; all configuration is read from the supplied properties file.

# Improvements
1. **Add transactional safety** – Perform INSERT into a staging table, verify row count, then swap tables or rename to avoid empty target on failure.
2. **Parameterize month selection** – Accept start/end month arguments instead of hard‑coded `current_day` checks, enabling flexible back‑fill and reducing maintenance.