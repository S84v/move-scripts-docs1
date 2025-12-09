# Summary
`SIMActivesDataAccess` implements the data‑access layer for the SIM Inventory loader. It retrieves “actives” records from Hadoop (Impala), transforms them into `SIMInvRecord` DTOs, and upserts them into the Oracle `MOVE_SIM_INVENTORY_STATUS` table. It also provides utilities to refresh service‑abbreviation flags and to query the latest insert‑time/partition information from the raw Hadoop tables. All operations are logged via Log4j and wrapped in explicit transaction handling.

# Key Components
- **Class `SIMActivesDataAccess`** – central DAO for SIM inventory.
- **`updateSIMInventory(List<SIMInvRecord>)`** – batch upsert (insert or update) of actives into Oracle; returns duplicates for re‑processing.
- **`fetchActives(Timestamp start, Timestamp end, boolean isBacklog)`** – pulls actives from Impala using parameterised SQL from `MOVEDAO.properties`.
- **`updateServAbbrs()`** – executes three hard‑coded update statements to set missing service abbreviations (IoT, SIM, MVNE).
- **`getMaxInsertDateTime()`** – returns the maximum `inserttime` from the Hadoop raw table (used to compute next batch window).
- **`getLatestPartitonDate()`** – returns the maximum partition date from the Hadoop raw table.
- **`getStackTrace(Exception)`** – utility to convert an exception stack trace to a `String` for logging.

# Data Flow
| Method | Input | Process | Output / Side‑Effect |
|--------|-------|---------|----------------------|
| `updateSIMInventory` | `List<SIMInvRecord>` (actives) | Load existing SIM keys from Oracle, decide insert vs update, batch statements (size = 2000), commit, rollback on error | Returns `List<SIMInvRecord>` containing duplicates; Oracle table updated/inserted |
| `fetchActives` | `Timestamp startDate`, `Timestamp endDate`, `boolean isBacklog` | Build SQL from `MOVEDAO`, bind dates, execute on Impala, map rows to `SIMInvRecord` | `List<SIMInvRecord>` |
| `updateServAbbrs` | none | Retrieve three update SQL strings from `MOVEDAO`, execute sequentially on Oracle | Rows updated in Oracle |
| `getMaxInsertDateTime` | none | Execute `max.actives.inserttime` query on Impala | `Timestamp` (max insert time) |
| `getLatestPartitonDate` | none | Execute `max.active.partition` query on Impala | `String` (max partition date) |

External services & resources:
- **Oracle DB** – via `JDBCConnection.getOracleConnection()`.
- **Impala/Hadoop** – via `JDBCConnection.getImpalaConnection()`.
- **`MOVEDAO.properties`** – stores all SQL statements.
- **Log4j** – logging of progress, errors, and stack traces.

# Integrations
- **`JDBCConnection`** – central connection manager (reads system properties populated by `MNAAS_ShellScript.properties`).
- **`SIMInvRecord` DTO** – shared across the Move mediation pipeline (used by loader, downstream processors, and audit components).
- **`log4j.properties`** – defines logging destinations for this DAO.
- **Batch driver** – higher‑level job (e.g., `SIMInvLoaderMain`) invokes `fetchActives` → `updateSIMInventory` in a loop based on `getMaxInsertDateTime`.

# Operational Risks
- **Large in‑memory collections** – `sims.contains()` and `idsToInsert.contains()` are O(n); may cause memory pressure on high‑volume days. *Mitigation*: replace with `HashSet<String>`.
- **Fixed batch size (2000)** – may not align with Oracle/Impala limits; could cause `ORA‑01795` or network timeouts. *Mitigation*: make batch size configurable.
- **Transaction handling** – only Oracle connection is rolled back; Impala reads are not transactional but may be stale if failures occur after partial inserts. *Mitigation*: ensure idempotent upserts or use staging tables.
- **Resource leaks** – manual close in finally blocks; if an exception occurs before `connection` is assigned, `closeOracleConnection()` may be called on a null reference. *Mitigation*: use try‑with‑resources for `PreparedStatement`, `ResultSet`, and `Connection`.
- **Hard‑coded column names** – schema changes break the DAO at compile time. *Mitigation*: externalize column mappings or use ORM.

# Usage
```java
public static void main(String[] args) throws Exception {
    SIMActivesDataAccess dao = new SIMActivesDataAccess();

    // Determine window (example: last hour)
    Timestamp end = dao.getMaxInsertDateTime();
    Calendar cal = Calendar.getInstance();
    cal.setTime(end);
    cal.add(Calendar.HOUR, -1);
    Timestamp start = new Timestamp(cal.getTimeInMillis());

    // Fetch actives from Hadoop
    List<SIMInvRecord> actives = dao.fetchActives(start, end, false);

    // Upsert into Oracle
    List<SIMInvRecord> duplicates = dao.updateSIMInventory(actives);

    // Optional: re‑process duplicates
    if (!duplicates.isEmpty()) {
        dao.updateSIMInventory(duplicates);
    }

    // Refresh service abbreviations
    dao.updateServAbbrs();
}
```
For debugging, set Log4j level to `DEBUG` in `log4j.properties` and monitor the generated daily rolling log file.

# Configuration
- **System properties** (set by `MNAAS_ShellScript.properties`):
  - `oracle.url`, `oracle.user`, `oracle.password`
  - `impala.url`, `impala.user`, `impala.password`
- **`MOVEDAO.properties`** – contains keys:
  - `actives.insert`, `actives.update`, `actives`, `actives.backlog`
  - `sim.inventory.sim`, `iot.update`, `sim.update`, `mvne.update`
  -