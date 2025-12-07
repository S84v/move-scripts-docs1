# Summary
`MOVEDAO.properties` centralises all SQL statements used by the Move‑Mediation‑Billing‑Reports daily job. It provides SELECT queries for various usage, activation, tolling and addon reports, as well as DML for truncating and inserting month‑level aggregates into `mnaas.month_reports`. The DAO layer reads this file to build prepared statements that populate downstream CSV/Excel reports and feed downstream billing/analytics pipelines.

# Key Components
- **Query definitions** – property keys (e.g., `usage.fetch`, `tadig.usage.fetch`, `hol.active`) mapping to static SELECT statements.
- **DML statements** – `truncate` and `insert` for managing the `month_reports` staging table.
- **Report categories** – usage, SMS, tolling, addon, activation, eSIM, BYON, etc., each with a dedicated query.
- **Ordering clauses** – deterministic `ORDER BY` for reproducible result sets.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| DAO init | `MOVEDAO.properties` file (classpath) | Load key/value pairs into a `Properties` object | In‑memory map of query strings | None |
| Report generation | JDBC connection (configured via external DB URL/credentials) | Prepare statement using the query string; bind any runtime parameters (`?` placeholders only in `insert`) | `ResultSet` streamed to report writer (CSV, Excel, DB load) | Network I/O, DB cursor consumption |
| Month‑report aggregation | Same DB connection | Execute `truncate` → clear `mnaas.month_reports`; execute `insert` with six bound values (month, next_month, start/end timestamps, start/end dates) | Populated `month_reports` table | Table lock during truncate/insert |
| Auxiliary look‑ups (e.g., `all.ipvprobe`) | `month_reports` and `org_details` tables | Join and aggregate daily IPvProbe data | Aggregated rows for further processing | Reads from multiple tables |

External services: only the relational database (presumably Oracle/PostgreSQL) accessed via JDBC. No message queues or file I/O directly in this file.

# Integrations
- **DAO layer (`MoveDAO` or similar)** – reads this properties file, creates `PreparedStatement`s, and exposes methods such as `fetchUsage()`, `fetchHolActive()`, etc.
- **Report generators** – consume DAO methods to produce daily CSV/Excel files stored in the `DailyReport` output directory.
- **Batch scheduler (e.g., Spring Batch, cron)** – triggers the DAO‑driven job each night.
- **Downstream billing system** – consumes the generated reports or reads directly from `mnaas.month_reports`.

# Operational Risks
- **Unbounded result sets** – large tables may cause OOM if the consumer loads the entire `ResultSet` into memory. *Mitigation*: stream rows, apply pagination or date filters.
- **Schema drift** – column order in SELECT statements is hard‑coded; schema changes break downstream mapping. *Mitigation*: use column aliases and maintain versioned queries.
- **Missing indexes** – many `ORDER BY` clauses on non‑indexed columns can degrade performance. *Mitigation*: review execution plans, add composite indexes matching the order.
- **Hard‑coded truncation** – `truncate` removes all rows without archival. *Mitigation*: archive month data before truncation or switch to `DELETE` with retention policy.
- **No parameterisation** – all queries are static; any future filter will require code change. *Mitigation*: refactor to use named parameters.

# Usage
```bash
# Example: run the daily job from command line (assuming a Spring Boot wrapper)
java -Dspring.profiles.active=prod -jar move-mediation-billing-reports.jar \
    --report=usage.fetch \
    --month=2025-11
```
*Debug*: set logger to `DEBUG` for `com.move.dao` to see the loaded SQL and bound parameters.

# Configuration
- **Environment variables**: `DB_URL`, `DB_USER`, `DB_PASSWORD` (or equivalent Spring datasource properties).
- **Config files**: `MOVEDAO.properties` located under `src/main/resources/` and packaged into the JAR.
- **JVM options**: `-Dlog.level=DEBUG` for detailed DAO logging.

# Improvements
1. **Externalise queries to a version‑controlled SQL repository** – store each query in its own `.sql` file and load at runtime; enables easier review, syntax highlighting, and independent versioning.
2. **Introduce query parameterisation and pagination** – replace static SELECTs with templates that accept date ranges, `tcl_secs_id` filters, and `LIMIT/OFFSET` to control result‑set size and reduce memory pressure.