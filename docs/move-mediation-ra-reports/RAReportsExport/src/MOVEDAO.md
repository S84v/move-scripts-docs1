# Summary
`MOVEDAO.properties` defines all SQL statements used by the **RAReportsExport** batch component to retrieve daily, monthly, and ad‑hoc reporting data from the `mnaas` schema. The service loads this file at runtime, substitutes the positional `?` parameters with job‑supplied values (dates, months, bill periods, etc.), and executes the queries to produce CSV/TSV extracts for downstream consumption (data lake staging, email reports, reconciliation).

# Key Components
- **SQL query entries** – each key (e.g., `daily.file.fetch`, `monthly.reject.fetch`, `ppu.usg.fetch`) maps to a single prepared statement.
- **Prepared‑statement loader** – Java utility (not shown) reads the properties file, creates `PreparedStatement` objects, and caches them for reuse.
- **Parameter binding layer** – job scheduler passes concrete values (date strings, month strings, bill month, usage type) that replace the `?` placeholders before execution.

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | Runtime parameters supplied by the export job: `file_date`, `start_month`, `end_month`, `bill_month`, `usage_type`, etc. |
| **Processing** | `MOVEDAO.properties` → `Properties` object → `PreparedStatement` → bind parameters → execute against Hive/Impala (via JDBC). |
| **Output** | ResultSets streamed to CSV/TSV files written to local extract directories (configured in `MNAAS_ShellScript.properties`). |
| **Side Effects** | None beyond DB reads; files are later uploaded via SFTP by shell scripts. |
| **External Services** | Hive/Impala cluster (`mnaas` schema), file system (local staging), SFTP target (data lake). |

# Integrations
- **`MNAAS_ShellScript.properties`** – provides Hive/Impala connection strings and authentication used by the Java component that consumes these queries.
- **`log4j.properties`** – logs query execution start/end, row counts, and errors.
- **Shell scripts** (`run_ra_reports.sh` etc.) – invoke the Java export service, passing date/month arguments that are bound to the queries.
- **Email/notification module** – reads generated files and sends status reports; not directly referenced but part of the overall batch flow.

# Operational Risks
- **SQL syntax drift** – schema changes (e.g., column rename) break queries; mitigate with version‑controlled schema migration scripts and integration tests.
- **Parameter mismatch** – incorrect date format leads to empty results or runtime errors; enforce ISO‑8601 (`yyyy-MM-dd`) validation in the job scheduler.
- **Performance bottlenecks** – large `WITH` CTEs (monthly.geneva.fetch, ppu.usg.fetch) may cause long Hive scans; add appropriate partition filters and statistics.
- **Credential exposure** – DB credentials stored in `MNAAS_ShellScript.properties`; protect with OS‑level permissions and rotate secrets regularly.

# Usage
```bash
# Example: generate daily file count report for 2024‑11‑30
java -cp ra-reports-export.jar \
     com.move.raexport.RAExportMain \
     --query=daily.file.fetch \
     --param=2024-11-30 \
     --output=/data/extracts/daily_file_counts_20241130.csv
```
Debugging: set `log4j.rootLogger=DEBUG,stdout` in `log4j.properties` to view bound parameters and row counts.

# Configuration
- **`MOVEDAO.properties`** – this file (SQL definitions).
- **`MNAAS_ShellScript.properties`** – Hive/Impala JDBC URL, user/password, default directories.
- **Environment variables** (referenced by shell scripts): `HIVE_JDBC_URL`, `HIVE_USER`, `HIVE_PASSWORD`, `EXPORT_DIR`, `SFTP_HOST`, `SFTP_USER`, `SFTP_KEY`.
- **Log4j** – `logfile.name` property defines the path for audit logs.

# Improvements
1. **Parameter‑type metadata** – add a companion `.meta` file describing expected data types (date, int, string) for each query to enable automatic validation before binding.
2. **Query templating** – migrate to a lightweight templating engine (e.g., Apache Velocity) to reduce duplication of common CTEs and improve readability/maintainability.