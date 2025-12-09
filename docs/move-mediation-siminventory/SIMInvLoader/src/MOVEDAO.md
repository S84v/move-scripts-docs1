# Summary
`MOVEDAO.properties` is a runtime‑loaded property file that supplies all SQL statements used by the **SIMInvLoader** DAO layer. The queries drive extraction of active SIM records (full and backlog loads), maintenance of the `MOVE_SIM_INVENTORY_STATUS` inventory table, service‑abbreviation classification, loader‑log persistence, and file‑audit recording for the Move mediation pipeline.

# Key Components
- **DAO classes (Java/Scala)** – read this file, retrieve the named SQL strings, prepare statements, bind parameters, and execute against the Oracle `MNAAS` schema.  
- **SQL statements** – grouped by functional area:  
  - `actives`, `actives.backlog` – CTE‑based extracts from `mnaas.actives_raw_daily`.  
  - `max.actives.inserttime`, `max.active.partition` – housekeeping queries.  
  - `sim.inventory.*` – INSERT/UPDATE of `MOVE_SIM_INVENTORY_STATUS`.  
  - `iot.update`, `sim.update`, `mvne.update` – service‑abbreviation mass updates.  
  - `get.dates`, `date.insert`, `date.update` – loader‑log CRUD.  
  - `audit.insert`, `audit.update` – file‑audit CRUD.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | Date range parameters (batch start/end) | `actives` / `actives.backlog` SELECT | Result set of active SIM rows streamed to the loader. |
| 2 | Result rows | DAO maps columns → inventory fields | Calls `sim.inventory.insert` or `sim.inventory.update`. |
| 3 | Inventory table state | Periodic `iot.update`, `sim.update`, `mvne.update` | Bulk `serv_abbr` classification updates. |
| 4 | Loader job identifiers | `get.dates` → `date.insert` / `date.update` | Records batch start/end and status in `Move_loader_log`. |
| 5 | Processed file metadata | `audit.insert` / `audit.update` | Persists file‑audit rows in `move_sim_inventory_file_audit`. |
| External services | Oracle `MNAAS` schema (read/write) | JDBC/OCI driver | No external queues; all operations are synchronous DB calls. |

# Integrations
- **MNAAS_ShellScript.properties** – supplies Oracle connection URL, user, password, and batch size values used by the DAO at startup.  
- **log4j.properties** – provides logging for DAO execution (INFO/ERROR).  
- **Shell scripts / Spark jobs** – invoke the Java/Scala loader, which loads this property file to obtain the SQL.  
- **Move_loader_log** – consumed by monitoring/alerting scripts to detect job health.  
- **move_sim_inventory_file_audit** – read by downstream reporting jobs.

# Operational Risks
- **SQL syntax drift** – manual edits can introduce syntax errors; mitigate with CI linting of property files.  
- **NULL comparison (`lasttrafficdate <> NULL`)** – always evaluates to UNKNOWN; may cause missing rows. Replace with `IS NOT NULL`.  
- **Hard‑coded column order** – schema changes break INSERT/UPDATE bindings; enforce versioned schema contracts.  
- **Batch size mismatch** – large result sets may OOM the loader; tune `batch.size` in `MNAAS_ShellScript.properties`.  

# Usage
```bash
# Example: run the loader in debug mode
export LOG_LEVEL=DEBUG
java -cp siminvloader.jar \
  -Dconfig.file=../MNAAS_ShellScript.properties \
  com.move.siminvloader.SIMInvLoaderMain \
  --start "2024-10-01 00:00:00" \
  --end   "2024-10-01 23:59:59"
```
The main class loads `MOVEDAO.properties`, resolves the `actives` query, executes it with the supplied timestamps, and persists results via the inventory statements.

# Configuration
- **Environment variables**: `ORACLE_JDBC_URL`, `ORACLE_USER`, `ORACLE_PASSWORD` (referenced in `MNAAS_ShellScript.properties`).  
- **Config files**: `MOVEDAO.properties` (SQL), `MNAAS_ShellScript.properties` (connection & runtime constants), `log4j.properties` (logging).  

# Improvements
1. **Replace invalid NULL comparisons** – update `lasttrafficdate <> NULL` to `lasttrafficdate IS NOT NULL` in both `actives` and `actives.backlog` queries.  
2. **Externalize column lists** – move column enumerations to a separate metadata file and generate INSERT/UPDATE statements programmatically to stay in sync with schema changes.