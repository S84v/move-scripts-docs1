# Summary
`MOVEDAO.properties` centralises all SQL statements required by the Move‑Mediation‑Billing‑Reports daily job. The DAO layer loads this file, creates prepared statements for a variety of usage, activation, tolling and add‑on reports, and executes DML to refresh the `mnaas.month_reports` aggregate table. Results are streamed to CSV/Excel files and downstream billing/analytics pipelines.

# Key Components
- **SQL query keys** – property names (e.g., `usage.fetch`, `hol.active`) that map to SELECT statements for specific report dimensions.  
- **DML keys** – `truncate` and `insert` statements used to rebuild the `mnaas.month_reports` table each run.  
- **DAO loader** – Java (or shell) component that reads the properties file, substitutes `?` placeholders, and prepares `PreparedStatement` objects.  
- **Report generators** – modules that consume the result sets, format rows, and write to CSV/Excel.  

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1. Load config | `MOVEDAO.properties` file (classpath) | `java.util.Properties.load()` | In‑memory map of query keys → SQL strings | None |
| 2. Build statements | DB connection parameters from `MNAAS_ShellScript.properties` | `Connection.prepareStatement(sql)` | `PreparedStatement` objects | None |
| 3. Execute SELECTs | Prepared statements, optional bind values | `ResultSet rs = stmt.executeQuery()` | Row streams fed to CSV/Excel writers | DB read load |
| 4. Refresh aggregates | `truncate` then `insert` statements | `stmt.executeUpdate()` | Updated rows in `mnaas.month_reports` | Table rewrite (transactional) |
| 5. Notification | Success/failure status | Email via parameters in `MNAAS_ShellScript.properties` | Status email | External SMTP usage |

# Integrations
- **MNAAS_ShellScript.properties** – supplies Impala host/port, JDBC URL, authentication, and email routing; consumed by the DAO startup script.  
- **log4j.properties** – provides logging for DAO execution, query timing, and error capture.  
- **Shell/Java orchestration scripts** – invoke the DAO layer, pass the property file path, and trigger CSV/Excel export utilities.  
- **Downstream pipelines** – generated CSV/Excel files are ingested by billing/analytics jobs (e.g., Hadoop, Spark).  

# Operational Risks
- **Stale/invalid SQL** – schema changes break queries; mitigate with automated regression test suite that validates each query against the current Impala schema.  
- **Uncontrolled growth of result sets** – large SELECTs may exhaust memory; mitigate by streaming `ResultSet` directly to file and applying pagination where possible.  
- **Missing truncate/insert ordering** – if `truncate` fails, `insert` may append to stale data; mitigate by wrapping both statements in a single transaction (or using `DROP/CREATE` pattern).  
- **Configuration drift** – divergence between this file and `MNAAS_ShellScript.properties`; mitigate with CI linting that checks for referenced keys.  

# Usage
```bash
# Example Java invocation
java -cp lib/*:conf/ \
    com.mnaas.reports.DAORunner \
    -props conf/MOVEDAO.properties \
    -env conf/MNAAS_ShellScript.properties \
    -report usage.fetch
```
*Debug*: set `log4j.rootLogger=DEBUG, stdou` in `log4j.properties` and run with `-Dlogfile.name=debug.log`.

# configuration
- **Environment variables**: `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` (referenced in `MNAAS_ShellScript.properties`).  
- **Config files**: `MNAAS_ShellScript.properties` (DB connection, email), `log4j.properties` (logging).  
- **Runtime parameters**: report key (`usage.fetch`, `hol.active`, etc.), optional bind values for the `insert` statement.  

# Improvements
1. **Externalise query versioning** – store SQL in a version‑controlled database table and load at runtime to avoid redeployments for query tweaks.  
2. **Add query metadata** – augment each property with a comment block containing expected column types, row limits, and owner contact to aid maintenance and impact analysis.