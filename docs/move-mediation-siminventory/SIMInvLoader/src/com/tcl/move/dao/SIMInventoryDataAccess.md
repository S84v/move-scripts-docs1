# Summary
`SIMInventoryDataAccess` implements the DAO for the Move mediation SIM inventory loader. It reads SQL statements from `MOVEDAO.properties`, upserts SIM records into the Oracle `MOVE_SIM_INVENTORY_STATUS` table, marks SIMs as pre‑active, validates creation dates against the file date, and parses raw diff strings into `SIMInvRecord` DTOs. All operations are logged and executed within a single transactional scope.

# Key Components
- **Class `SIMInventoryDataAccess`**
  - `insertUpdateSIMInventory(...)` – core upsert routine; handles pre‑active updates, batch inserts/updates, transaction commit/rollback.
  - `checkCreationDate(...)` – filters records whose creation date does not match the file date ± 1 day.
  - `getRoundedDate(Date)` – normalizes a `Date` to midnight.
  - `fetchSIMInventory(List<String>)` – parses semi‑colon delimited raw records into `SIMInvRecord` objects.
  - `getStackTrace(Exception)` – utility to convert stack trace to string for logging.
- **ResourceBundle `daoBundle`** – loads SQL strings from `MOVEDAO.properties`.
- **JDBCConnection** – static helper providing Oracle connections and cleanup.
- **Constants / fields**
  - `formatter` – `SimpleDateFormat` for parsing creation timestamps.
  - `preactiveCount` – system property controlling max SIMs allowed to be moved to pre‑active in a single run.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | System properties (`siminventory_preactive_count`, region‑specific time‑zone) | Loaded into `preactiveCount` and used for batch size checks. |
| 2 | `List<SIMInvRecord> simInvListNew`, `List<SIMInvRecord> simInvListRemoved`, `region`, `fileDate` | `insertUpdateSIMInventory` obtains Oracle connection, prepares statements, disables auto‑commit. |
| 3 | `simInvListRemoved` (optional) | Executes `UPDATE ... PREACTIVE` either for all SIMs in region or per‑SIM batch; throws `FileException` if count exceeds threshold. |
| 4 | `simStatement` query | Retrieves all existing SIM identifiers from `MOVE_SIM_INVENTORY_STATUS` into `sims`. |
| 5 | `simInvListNew` (filtered by `checkCreationDate`) | For each record: if SIM exists → add to update batch; else → add to insert batch. Batches executed every 1 000 rows. |
| 6 | After loop | Executes any remaining batches, commits transaction. |
| 7 | Exception path | Rolls back transaction, logs stack trace, re‑throws `DBDataException` or `DBConnectionException`. |
| 8 | Finally block | Closes statements, result set, and Oracle connection. |
| 9 | `fetchSIMInventory` | Parses raw diff strings (semicolon delimited) into `SIMInvRecord` objects; validates status; logs parsing errors. |

External services:
- Oracle DB (via JDBC)
- System properties (environment configuration)
- Log4j for logging

# Integrations
- **`MOVEDAO.properties`** – supplies SQL strings referenced by keys: `sim.inventory.updateall`, `sim.inventory.preactive`, `sim.inventory.insert`, `sim.inventory.update`, `sim.inventory.sim`.
- **`JDBCConnection`** – shared connection manager used by other DAO classes (`SIMActivesDataAccess`, etc.).
- **`SIMInvRecord` DTO** – also used by `SIMActivesDataAccess` and downstream loader components.
- **Shell scripts / job scheduler** – set system properties (e.g., `siminventory_preactive_count`, region‑specific time‑zone) before invoking the Java job.

# Operational Risks
- **Batch size overflow** – `simInvListRemoved` exceeding `preactiveCount` aborts the entire run. Mitigation: monitor input size, adjust property, or implement chunked processing.
- **Memory pressure** – loading all existing SIMs into `List<String> sims` may be large for high‑volume regions. Mitigation: stream existence check via indexed query or use temporary table.
- **Date validation strictness** – records with creation dates ± 1 day are dropped silently; could cause data loss. Mitigation: add alerting/reporting for filtered records.
- **Hard‑coded batch size (1000)** – may not align with DB/driver limits. Mitigation: make batch size configurable.
- **Connection leak on exception before `connection` assignment** – `connection` may be null in catch block; rollback attempt could NPE. Mitigation: guard rollback with null check.

# Usage
```bash
# Set required system properties
export siminventory_preactive_count=500
export REGION=EMEA
export EMEA=UTC

# Run the loader (example entry point class)
java -cp <classpath> com.tcl.move.loader.SIMInventoryJob \
     -region EMEA \
     -fileDate 2024-12-01 \
     -newSimFile /data/sim/new_sim.txt \
     -removedSimFile /data/sim/removed_sim.txt
```
- Debug by attaching a remote JVM debugger to the process or by enabling Log4j DEBUG level for `com.tcl.move.dao.SIMInventoryDataAccess`.

# Configuration
- **Environment / System Properties**
  - `siminventory_preactive_count` – max SIMs allowed to be marked pre‑active per run.
  - Region‑specific time‑zone property (e.g., `EMEA=UTC`).
- **Config Files**
  - `MOVEDAO.properties` – SQL statement definitions.
  - `MNAAS_ShellScript.properties` – supplies JDBC URLs, usernames, passwords used by `JDBCConnection`.
- **Log4j configuration** – `log4j.properties` controlling logger output.

# Improvements
1. **Replace full SIM list load with existence check via indexed query** – use `SELECT SIM FROM MOVE_SIM_INVENTORY_STATUS WHERE REGION=? AND SIM IN (?)` or temporary staging table to avoid loading all SIMs into memory.
2. **Externalize batch size and date‑tolerance thresholds** – add properties `sim.inventory.batch.size` and `sim.inventory.date.tolerance.days` to `MOVEDAO.properties` or a dedicated config file, making the DAO adaptable without code changes.