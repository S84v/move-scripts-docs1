# Summary
`MOVEDAO.properties` is a property‑file that stores all SQL statements used by the **SIMInvLoader** component of the Move mediation pipeline. The queries drive extraction of active SIM records, incremental “backlog” loads, inventory inserts/updates, service‑abbreviation classification, loader‑log management and file‑audit persistence. The file is read at runtime by Java/Scala DAO classes to obtain prepared‑statement strings, which are then executed against the Oracle `MNAAS` schema.

# Key Components
- **actives** – CTE‑based query that returns the latest status per SIM (by `lastproductstatuschangedate`) and the latest traffic record (by `lasttrafficdate`) for a given insertion‑time window. Used for daily active‑SIM extraction.  
- **actives.backlog** – Similar to `actives` but filters on `lastproductstatuschangedate` / `lasttrafficdate` ranges, enabling re‑processing of historical windows.  
- **max.actives.inserttime** – Retrieves the maximum `inserttime` from `actives_raw_daily`; used to compute the next incremental window.  
- **max.active.partition** – Retrieves the maximum partition date; used for partition‑aware processing.  
- **sim.inventory.\*** – `insert`, `update`, `preactive`, `updateall`, `sim` statements that maintain the `MOVE_SIM_INVENTORY_STATUS` table (core inventory store).  
- **actives.\*** – `insert`, `update` statements that populate or amend inventory rows derived from the active‑SIM feed.  
- **iot.update / sim.update / mvne.update** – Classification updates that set `serv_abbr` based on `commercial_offer` values.  
- **get.dates / date.insert / date.update** – Loader‑log queries that track batch start/end timestamps and status per feed/app.  
- **audit.insert / audit.update** – File‑audit table DML for recording processing metadata (file name, start/end, record count, remarks).

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|-------------|-----------------------|
| 1 | `actives_raw_daily` rows (via JDBC) | `actives` / `actives.backlog` SELECTs with window functions | Result set of latest SIM status & traffic |
| 2 | Result set from (1) | DAO maps rows → inventory fields | Calls `sim.inventory.insert` or `actives.insert` |
| 3 | Existing inventory rows | `sim.inventory.update` / `actives.update` (if SIM present) | Updated `MOVE_SIM_INVENTORY_STATUS` rows |
| 4 | Post‑load classification | `iot.update`, `sim.update`, `mvne.update` | `serv_abbr` column populated |
| 5 | Loader execution context | `get.dates` → determine last successful batch | Batch window for step 1 |
| 6 | After successful batch | `date.insert` (new batch) or `date.update` (status) | `Move_loader_log` entry |
| 7 | File processing metadata | `audit.insert` at start, `audit.update` at end | `move_sim_inventory_file_audit` entry |

External services: Oracle database (`MNAAS` schema). No message queues; all interactions are synchronous JDBC calls.

# Integrations
- **SIMInvLoader Java driver** reads this file via `java.util.Properties` and supplies the SQL strings to `PreparedStatement` objects.  
- **Move_loader_log** and **move_sim_inventory_file_audit** tables are consumed by monitoring dashboards and downstream ETL jobs that rely on batch timestamps.  
- **VAZ_Daily_Aggr_Report** and other Move pipelines reference the same loader‑log table to determine incremental windows, establishing cross‑pipeline dependency.  
- **log4j.properties** (in the same module) provides logging for DAO execution; errors propagate to the loader‑log status field.

# Operational Risks
- **SQL injection risk** – Queries are parameterised (`?` placeholders); ensure DAO never concatenates user input.  
- **Stale window calculation** – If `max.actives.inserttime` returns `NULL` (e.g., empty source), subsequent window bounds become invalid → guard with default dates.  
- **Concurrent batch runs** – Simultaneous loaders could race on `Move_loader_log` rows; enforce unique `(feed, app, batch_start)` constraint or use DB locking.  
- **Data loss on failed update** – `sim.inventory.update` overwrites rows; missing `WHERE` clause on `REGION` could affect other regions → verify region filter is always supplied.  
- **Partition skew** – `max.active.partition` may lag behind actual data; monitor partition freshness to avoid missing late‑arriving records.

# Usage
```bash
# Example: run SIMInvLoader (Java entry point)
export DB_URL=jdbc:oracle:thin:@//dbhost:1521/PROD
export DB_USER=move_user
export DB_PASS=********
java -cp siminvloader.jar com.move.siminvloader.Loader \
     --properties=move-mediation-siminventory/SIMInvLoader/bin/MOVEDAO.properties \
     --feed=actives \
     --app=SIMInvLoader
```
*Debug*: enable `log4j.logger.com.move.siminvloader=DEBUG,stdout` in `log4j.properties` to see the resolved SQL strings and bound parameters.

# Configuration
- **File path**: `move-mediation-siminventory/SIMInvLoader/bin/MOVEDAO.properties` (loaded at runtime).  
- **Environment variables** required by the loader: `DB_URL`, `DB_USER`, `DB_PASS`.  
- **Referenced config files**: `log4j.properties` (logging), `MOVEDAO.properties` itself (SQL), and any external property file that supplies feed‑specific parameters (e.g., date window placeholders).

# Improvements
1. **Externalise query templates** to a version‑controlled SQL repository and load via a templating engine (e.g., MyBatis) to enable compile‑time validation and easier refactoring.  
2. **Add explicit transaction boundaries** around the insert‑update‑classification sequence and ensure `date.update` is performed in a single atomic transaction to prevent log inconsistency on partial failures.