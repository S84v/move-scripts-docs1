# Summary
`MOVEDAO.properties` is a property‑file repository of named SQL statements used by the **MoveGenevaInterface** component to read, aggregate, and write billing‑related data from Hadoop/Hive and Oracle sources. In production the file drives data‑extraction for activation, usage, CDR, and reject‑handling processes that feed the Geneva billing reconciliation workflow.

# Key Components
- **SQL query entries** (e.g., `activations.fetch`, `data.fetch`, `cdr.subset.insert`, `sim.reject.insert`) – each key maps to a parametrised Hive/Oracle statement executed by the DAO layer.  
- **Parameter placeholders** – `?` for positional bind variables, `{0}`, `{1}` … for dynamic fragments injected by the calling service.  
- **Category groups** – *Activation*, *Usage*, *Data*, *SMS/Voice*, *Traffic*, *Month‑Billing*, *Reject/Metadata* – logical grouping that mirrors the DAO service methods.

# Data Flow
| Stage | Input (bind vars / fragments) | Process | Output / Side‑Effect |
|-------|------------------------------|---------|----------------------|
| Extraction | Date range, `tcl_secs_id`, commercial offers, dynamic `IN (…)` lists | Hive/Oracle SELECT statements (e.g., `activations.fetch`, `data.zone.fetch`) | Result sets streamed to Java DTOs for further aggregation |
| Transformation | Result sets from extraction | In‑memory aggregation (SUM, COUNT, NVL) performed by SQL or downstream Java | Aggregated metrics (SIM counts, usage volumes) |
| Load / Persist | Aggregated values, generated IDs (`next.fileid`, `next.callid`) | Oracle INSERT/UPDATE statements (e.g., `sim.insert`, `usage.update`, `reject.update`) | Rows written to `sim_geneva`, `usage_geneva`, `geneva_reject`, `gen_file_cdr_mapping` |
| Cleanup | Temporary tables (`mnaas.billing_traffic_filtered_cdr`) | `truncate` statements | Tables cleared for next batch |

External services: Hive/Impala (for `mnaas.*` tables), Oracle DB (for `geneva_*` and `month_billing` tables). No message‑queue interaction is defined in this file.

# Integrations
- **MoveGenevaDAO** (Java class) reads this file via `Properties` loader, prepares `PreparedStatement`s, substitutes `{n}` fragments, and executes against the configured data sources.  
- **MoveGenevaService** orchestrates batch steps (extract → transform → load) and supplies runtime parameters (dates, IDs, offer lists).  
- **Batch Scheduler** (e.g., Spring Batch, Quartz) triggers the service; the properties file is the sole source of SQL definitions, enabling version‑controlled query changes without code recompilation.

# Operational Risks
- **SQL injection via `{n}` fragments** – dynamic fragments are concatenated; improper sanitisation can lead to injection. *Mitigation*: validate and whitelist allowed tokens before substitution.  
- **Schema drift** – queries reference Hive/Oracle tables/columns; schema changes break execution. *Mitigation*: maintain schema‑version compatibility matrix and automated integration tests.  
- **Performance bottlenecks** – large `GROUP BY` and `JOIN` operations on massive CDR tables. *Mitigation*: ensure proper partitioning, statistics, and consider incremental materialised views.  
- **Hard‑coded literals** (e.g., `'Data'`, `'SMS'`) may become outdated. *Mitigation*: externalise as configurable enums if business rules evolve.

# Usage
```bash
# Example: run a single extraction via the DAO utility
java -cp move-geneva.jar com.example.move.dao.MoveGenevaDAO \
     --query=activations.fetch \
     --param=2023-01-01 \
     --param=2023-01-31
# Debug: set log level to DEBUG in log4j.properties to see the final SQL after fragment substitution.
```
In IDE, set breakpoints in `MoveGenevaDAO.getQuery(String key)` to inspect the assembled statement.

# Configuration
- **Environment variables**: `HIVE_JDBC_URL`, `ORACLE_JDBC_URL`, `HIVE_USER`, `ORACLE_USER`, `HIVE_PASSWORD`, `ORACLE_PASSWORD`.  
- **External config files**: `application.properties` (holds DB connection pools), `log4j.properties` (logging).  
- **File location**: `move-mediation-billing/MoveGenevaInterface/bin/default/MOVEDAO.properties` – loaded on classpath under `resources/`.

# Improvements
1. **Parameterisation of dynamic fragments** – replace `{n}` string concatenation with a safe templating engine (e.g., Apache Velocity) that validates allowed tokens.  
2. **Query versioning** – move each logical group into separate `.sql` files and reference them via a manifest, enabling easier diffing and selective redeployment.