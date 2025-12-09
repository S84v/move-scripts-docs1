# Summary
`MOVEDAO.properties` is a configuration file that stores the SQL statements used by the **RAReportsExport** component of the MOVE‑RA‑Reports subsystem. The queries retrieve daily, monthly, and ad‑hoc reporting data (file counts, CDR usage, reject details, add‑on statistics, IPv probe reconciliation, PPU usage, etc.) from the `mnaas` schema. At runtime the export service loads this file, substitutes the positional parameters (`?`) with values supplied by the job (dates, months, bill periods, usage types), executes the statements against the production Oracle/SQL‑Server database, and writes the result sets to CSV/Excel files for downstream analytics and audit.

---

# Key Components
- **`MOVEDAO.properties`** – Plain‑text key/value store; each key is a logical query identifier, the value is the full SQL string.  
  - Keys follow a naming convention: `<frequency>.<entity>.fetch` (e.g., `daily.file.fetch`, `monthly.reject.fetch`).  
  - Queries use JDBC‑style positional placeholders (`?`) for runtime binding.  
- **Consumers (outside this file)**  
  - `RAReportsExport` Java class (or equivalent batch driver) that loads the properties via `java.util.Properties`.  
  - DAO layer (`MOVEDAO` or similar) that maps each key to a `PreparedStatement`, sets parameters, and streams the `ResultSet`.  

---

# Data Flow
| Step | Description |
|------|-------------|
| **1. Input** | Execution context supplies parameters: dates (`java.sql.Date`), month strings (`YYYY‑MM`), bill month, usage type, etc. |
| **2. Load** | `MOVEDAO.properties` is read from the classpath (`bin/`). The `Properties` object holds the raw SQL strings. |
| **3. Prepare** | For each required report, the DAO retrieves the SQL by key, creates a `PreparedStatement`, and binds the supplied parameters to the `?` placeholders. |
| **4. Execute** | The statement runs against the `mnaas` database (Oracle/SQL‑Server). |
| **5. Output** | Result sets are transformed into CSV/Excel files written to the configured output directory, or streamed to downstream services (e.g., HDFS, S3). |
| **6. Side‑effects** | None beyond DB reads and file writes; no mutations of source tables. |

**External Services / DBs**
- **Database**: `mnaas` schema (production data warehouse).  
- **File System**: Output directory defined elsewhere (e.g., via `logfile.name` or job parameters).  

---

# Integrations
- **`RAReportsExport` batch job** – Invokes the DAO, passing the appropriate keys (`daily.file.fetch`, `monthly.geneva.fetch`, etc.) based on the report type requested.  
- **Logging** – Log4j configuration (`log4j.properties`) captures query execution times and any errors.  
- **Scheduler / Orchestration** – Cron, Control‑M, or Airflow jobs trigger the export with date arguments that map to the placeholders in these queries.  
- **Downstream analytics** – Generated CSV/Parquet files are consumed by BI tools (Tableau, PowerBI) or data pipelines (Spark, Hive).  

---

# Operational Risks
| Risk | Impact | Mitigation |
|------|--------|------------|
| **SQL syntax errors / schema changes** | Job failures, missing data | Version‑control the properties file; add integration tests that validate each query against a test DB schema. |
| **Parameter mismatch** (wrong number/type) | `SQLException: Invalid column index` | Centralize parameter mapping in DAO; enforce strict method signatures per query key. |
| **Performance degradation** (large scans) | Long runtimes, resource contention | Add appropriate indexes on filter columns (`file_date`, `bill_month`, `usage_type`). Use query hints or materialized views for heavy aggregations. |
| **Credentials leakage** | Unauthorized DB access | Store DB credentials in a secure vault; do not embed them in this file. |
| **File system I/O saturation** | Incomplete or corrupted output files | Write to a staging area, then atomically move to final location; monitor disk usage. |

---

# Usage
```bash
# Example: Run daily file count report for 2024‑11‑30
java -cp lib/*:bin \
    com.tcl.raexport.RAReportsExport \
    --reportKey daily.file.fetch \
    --date 2024-11-30 \
    --output /data/ra/reports/daily_file_20241130.csv
```
*Debugging*:  
1. Enable `log4j.logger.com.tcl.raexport=DEBUG` to see the final SQL after parameter substitution.  
2. Use a DB client to execute the raw query from the properties file with sample parameters to verify results.  

---

# Configuration
- **File**: `move-mediation-ra-reports/RAReportsExport/bin/MOVEDAO.properties` (must be on the classpath).  
- **Environment Variables** (referenced elsewhere, not in this file):  
  - `DB_URL`, `DB_USER`, `DB_PASSWORD` – database connection.  
  - `OUTPUT_DIR` – base directory for generated reports.  
  - `logfile.name` – defined in `log4j.properties` for logging output.  

---

# Improvements
1. **Externalize Queries to a Version‑Controlled Repository** – Move the SQL statements to a dedicated `sql/` directory with `.sql` files and load them at runtime. This improves readability, enables IDE syntax checking, and simplifies diffing of query changes.  
2. **Add Query Metadata** – Include a comment block or separate properties (e.g., `<key>.description`, `<key>.timeout`) to document purpose, expected parameter order, and performance expectations. This aids maintainability and reduces runtime errors.