# Summary
`MOVEDAO.properties` defines the parameterized SQL statements used by the **APISimCountLoader** Java component to manage monthly and yearly move‑mediation SIM inventory and activity count tables in the `mnaas` schema. The statements retrieve partition dates, truncate target tables, insert aggregated counts, and handle partition lifecycle for production data pipelines.

# Key Components
- **part.date.raw** – Query to obtain the latest `partition_date` from `move_sim_inventory_status`.
- **part.date.aggr** – Query to obtain the latest `partition_date` from `move_sim_inventory_count`.
- **part.date.month** – Query to obtain the latest `status_date` from `move_curr_month_count`.
- **truncate.month** – Truncates the `move_curr_month_count` table.
- **insert.month.active** – Inserts aggregated active SIM counts for the current month.
- **insert.month.activity** – Inserts distinct SIM activity counts for the current month (parameterized date range).
- **insert.lastmonth.active** – Inserts aggregated active SIM counts for the previous month.
- **drop.part.year** – Drops a yearly partition identified by `{0}`.
- **insert.year.active** – Inserts aggregated active SIM counts for a specific year partition (parameterized date and partition key).
- **insert.year.activity** – Inserts distinct SIM activity counts for a specific year partition (parameterized date range and partition key).
- **get.partiton.count** – Retrieves the count of distinct yearly partitions.
- **get.min.part** – Retrieves the earliest yearly partition value.

# Data Flow
- **Inputs**: Runtime parameters (date ranges, partition keys) supplied by the Java loader; environment‑derived JDBC connection string from `MNAAS_ShellScript.properties`.
- **Processing**: Java component reads `MOVEDAO.properties`, prepares statements, substitutes parameters, executes against Impala/Hive.
- **Outputs**: Populated rows in `move_curr_month_count` and `move_curr_year_count`; updated partition metadata.
- **Side Effects**: Table truncation, partition drop, data insertion; potential data loss if truncation runs unintentionally.
- **External Services/DBs**: Impala/Hive cluster (`mnaas` schema); no message queues.

# Integrations
- **MNAAS_ShellScript.properties** – Provides Impala host and JDBC port for connection URL.
- **log4j.properties** – Controls logging for the loader execution.
- **APISimCountLoader Java code** – Loads this property file, creates `PreparedStatement` objects, and orchestrates execution order.
- **Shell scripts** – May invoke the Java JAR with classpath that includes this properties file.

# Operational Risks
- **Data loss**: `truncate.month` removes all current‑month data; must run only after successful backup.
- **Partition mis‑management**: Incorrect `{0}` substitution in `drop.part.year` can drop unintended partitions.
- **Date handling errors**: Use of `now()` and `trunc()` functions may produce off‑by‑one errors around month boundaries.
- **SQL injection**: Direct string substitution for `{0}` without proper escaping could be exploited.
- **Resource contention**: Large `INSERT … SELECT` operations may saturate Impala resources.

# Usage
```bash
# Set environment for Impala connection
export IMPALAD_HOST=10.0.1.5
export IMPALAD_JDBC_PORT=21050

# Run the loader (example command)
java -cp apisimloader.jar com.example.apisimloader.APISimCountLoader \
    --config=move-mediation-apisimloader/APISimCountLoader/src/MOVEDAO.properties \
    --dateFrom=2024-10-01 --dateTo=2024-10-31 --yearPartition=2024
```
- Adjust `--dateFrom`, `--dateTo`, and `--yearPartition` as required.
- Verify logs in the configured log file for success/failure.

# configuration
- **Environment Variables**: `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` (referenced by `MNAAS_ShellScript.properties`).
- **Config Files**: `MOVEDAO.properties` (SQL statements), `MNAAS_ShellScript.properties` (JDBC endpoint), `log4j.properties` (logging).
- **JDBC URL**: Constructed as `jdbc:impala://{IMPALAD_HOST}:{IMPALAD_JDBC_PORT}/default;AuthMech=3`.

# Improvements
1. Replace inline `{0}` placeholders with named parameters and use `PreparedStatement` to prevent injection and improve readability.
2. Externalize date calculations to the Java layer; pass explicit start/end dates to all queries to avoid reliance on Impala `now()` and `trunc()` functions.