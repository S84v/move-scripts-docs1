# Summary
`MOVEDAO.properties` defines a collection of parameterized SQL statements used by the APISimCountLoader component to manage and populate move‑mediation SIM inventory and activity count tables (monthly and yearly partitions) in the `mnaas` schema. The statements support retrieving the latest partition dates, truncating target tables, inserting aggregated SIM counts, and managing partition lifecycle.

# Key Components
- **SQL query definitions**
  - `part.date.raw` – Retrieves the latest raw inventory partition date.
  - `part.date.aggr` – Retrieves the latest aggregated inventory partition date.
  - `part.date.month` – Retrieves the latest month‑level status date.
  - `truncate.month` – Clears the current month count table.
  - `insert.month.active` – Inserts active SIM counts per business unit for the current month.
  - `insert.month.activity` – Inserts activity‑type SIM counts for a given date range (place‑holders `?` and `{0}`).
  - `insert.lastmonth.active` – Inserts active SIM counts for the previous month.
  - `drop.part.year` – Drops a yearly partition identified by `{0}`.
  - `insert.year.active` – Inserts active SIM counts for a specific year partition (`?` for date, `{0}` for partition key).
  - `insert.year.activity` – Inserts activity‑type SIM counts for a yearly partition (`?` for date range, `{0}` for partition key).
  - `get.partiton.count` – Returns the number of distinct yearly partitions.
  - `get.min.part` – Returns the earliest yearly partition key.

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| 1 | No direct input (triggered by loader schedule) | Execute `part.date.*` queries to determine latest partitions. | Latest partition timestamps. |
| 2 | `truncate.month` | Truncate `move_curr_month_count`. | Table emptied. |
| 3 | `insert.month.active` | Aggregate `move_sim_inventory_count` for current month (date range computed via `now()`). | Rows inserted per `buss_unit_id`/`prod_status`. |
| 4 | `insert.month.activity` | Aggregate distinct SIM activity from `traffic_details_raw_daily` for supplied date range (`?`). | Activity rows inserted. |
| 5 | `insert.lastmonth.active` | Aggregate previous month data. | Rows inserted for prior month. |
| 6 | Yearly partition maintenance (`drop.part.year`, `insert.year.*`) | Drop stale partition, insert active and activity rows for target year (`{0}`). | Yearly partition refreshed. |
| 7 | `get.partiton.count`, `get.min.part` | Query partition metadata. | Counts / min partition key for monitoring. |

External services:
- **Database**: Hive/Impala (or compatible) accessed via JDBC/ODBC; schema `mnaas`.
- **Scheduler**: Invokes loader (e.g., cron, Airflow) that reads this properties file.

# Integrations
- **APISimCountLoader Java component** reads `MOVEDAO.properties` to obtain SQL strings, substitutes placeholders (`?`, `{0}`) via `PreparedStatement` or string formatting, and executes them against the data warehouse.
- **Kafka / Notification subsystem** (unrelated to this file) supplies upstream SIM inventory data that populates `move_sim_inventory_count` and `traffic_details_raw_daily` tables.
- **Monitoring/Alerting**: Partition count queries (`get.partiton.count`) can feed health checks.

# Operational Risks
- **Partition key mismatch** – Incorrect `{0}` value may cause duplicate or missing partitions. *Mitigation*: Validate partition key format before execution.
- **Date range calculation errors** – Use of `now()` and `trunc` functions assumes server timezone consistency. *Mitigation*: Enforce UTC timezone in DB session.
- **Long‑running inserts** – Large aggregation may exceed query timeout. *Mitigation*: Tune Hive/Impala settings, add intermediate staging tables if needed.
- **Schema changes** – Adding/removing columns in source tables breaks insert statements. *Mitigation*: Version control of SQL and automated integration tests.

# Usage
```bash
# Example: run loader manually (Java class MainLoader)
java -cp loader.jar com.tcl.move.loader.APISimCountLoader \
    -config /path/to/MOVEDAO.properties \
    -date 2025-11-30   # optional date range for activity inserts
```
- The loader parses the properties file, substitutes `{0}` with the target partition identifier (e.g., `202511`), and executes the statements in the defined order.

# configuration
- **Environment variables**
  - `DB_JDBC_URL` – JDBC connection string to the `mnaas` warehouse.
  - `DB_USER` / `DB_PASSWORD` – Credentials.
- **Config files**
  - `MOVEDAO.properties` (this file) – SQL definitions.
  - `loader.conf` – Loader runtime parameters (batch size, retry count).

# Improvements
1. **Parameterize date calculations** – Replace inline `now()`/`trunc` logic with explicit `?` placeholders to allow deterministic testing and timezone control.
2. **Add checksum validation** – After each insert, run a count verification query and log discrepancies to enable automated data quality alerts.