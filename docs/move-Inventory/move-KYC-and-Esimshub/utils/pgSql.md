**File:** `move-Inventory\move-KYC-and-Esimshub\utils\pgSql.js`  

---

## 1. High‑Level Summary
`pgSql.js` encapsulates PostgreSQL connectivity and CDC‑related helper routines used by the KYC/Esim‑hub move scripts. It builds a connection configuration (single‑user or multi‑user mode) from the central `config` module, creates a `pg.Client` (or a `Pool` – currently commented out), and exports async functions for: opening/closing the connection, paging through source tables, pulling logical‑replication changes from a named slot, cleaning up processed LSNs, and extracting the latest LSN for checkpointing. All functions log their activity via the shared logger.

---

## 2. Important Classes / Functions

| Exported Symbol | Responsibility | Key Parameters | Returns / Side‑Effects |
|-----------------|----------------|----------------|------------------------|
| `connectPg()` | Opens a PostgreSQL connection and resolves with the connected client. | – | `Promise<Client>` – sets module‑level `pg_config` (the client). |
| `disconnectPg()` | Gracefully ends the client connection. | – | `Promise<void>` – logs closure. |
| `getRowsByPage(source, lastId, db, schema, table, pageSize, whereCond)` | Retrieves a page of rows from a regular table, using the primary‑key ordering to support incremental reads. | `source.pk` (comma‑separated PK list), `lastId` (array of last PK values), `schema`, `table`, `pageSize`, optional `whereCond`. | `{ rows: Array<object>, lastId: Array<any> }` – updates `lastId` to the PK of the last row returned. |
| `getRowsByPageDelta(slotName, lastId, limitSize)` | Reads CDC changes from a logical replication slot, optionally starting after a given LSN. | `slotName`, `lastId` (LSN string), `limitSize` (max rows). | `Array<object>` – raw rows from `pg_logical_slot_peek_changes`. |
| `removeProcessedLogs(slotName, lastId)` | Consumes (deletes) CDC entries that have been processed or ignored, based on global `processedLsn` / `ignoredLsn` arrays. | `slotName`, `lastId` (unused). | No return; executes `pg_logical_slot_get_changes` for the identified LSNs. |
| `getLsnSet(slot)` | Iteratively peeks into a slot, filters for changes belonging to the source table, and returns the most recent ≤ 1000 transactions. | `slot` (string). | `Array<{lsn:string,data:string}>`. |
| `getLsnSetOld(slot)` | Simple wrapper that returns *all* pending changes via `pg_logical_slot_peek_changes`. | `slot`. | `Array<object>`. |
| `getLatestLsn(lsnArray)` | Derives a checkpoint LSN (base‑64 encoded) from the last transaction(s) in a batch, handling possible “COMMIT” rows. | `lsnArray` (array of change objects). | `string` – encoded LSN. |

*Utility helpers*: `isObjFilled`, `mergeValues` – internal only.

---

## 3. Configuration & External Dependencies

| Source | Usage |
|--------|-------|
| `config` (`../config/config`) | Provides all PostgreSQL connection parameters (`pgsql.*`). Supports two modes: `pgsql.multiUser` (per‑table credentials) or single‑user. Also supplies optional SSL object (`pgsql.ssl`). |
| `logger` (`config.logMsg`) | Centralised logging (info, debug, error). |
| `globalDef` (`./globalDef`) | Imported but not referenced directly in this file; likely supplies shared globals such as `pgConnect`, `processedLsn`, `ignoredLsn`. |
| `pg` npm package (`pg.Client`) | PostgreSQL driver. The commented‑out `Pool` code indicates a future intention to use connection pooling. |
| Environment variables | Not accessed directly here; any env‑based overrides are expected to be resolved inside the `config` module. |

**Assumptions**

* A global variable `pgConnect` (or similar) is defined elsewhere (probably in `globalDef`) and is the active client used by all query functions.
* Arrays `processedLsn` and `ignoredLsn` are populated by other processors before calling `removeProcessedLogs`.
* The logical replication slot(s) already exist and are configured for the target database.

---

## 4. Inputs, Outputs, Side‑Effects, and Assumptions

| Function | Input | Output | Side‑Effect |
|----------|-------|--------|-------------|
| `connectPg` | – | Connected `Client` instance | Sets module‑level `pg_config` (client). |
| `disconnectPg` | – | `void` | Calls `client.end()`, clears `pgConnect`. |
| `getRowsByPage` | PK list, lastId, schema, table, pageSize, optional WHERE clause | `{rows, lastId}` | Executes a `SELECT … LIMIT`. |
| `getRowsByPageDelta` | Slot name, optional start LSN, optional limit | Array of CDC rows | Executes `pg_logical_slot_peek_changes`. |
| `removeProcessedLogs` | Slot name | – | Consumes CDC entries for LSNs in `processedLsn`/`ignoredLsn`. |
| `getLsnSet` | Slot name | Filtered transaction list (≤ 1000) | Multiple `pg_logical_slot_get_changes` calls. |
| `getLatestLsn` | Array of CDC rows | Base‑64 encoded LSN string | None (pure function). |

All DB interactions are **read‑only** except `removeProcessedLogs`, which consumes CDC entries (advancing the slot). No file system or external API calls are made.

---

## 5. Integration Points (How This Module Connects)

| Consumer | Expected Usage |
|----------|----------------|
| `processor/fullSync.js`, `processor/deltaSync/*.js` | Import `pgSql` to obtain a client (`connectPg`) and then call paging or CDC helpers for bulk load or change capture. |
| `utils/globalDef.js` | Likely defines the global `pgConnect`, `processedLsn`, `ignoredLsn` that this module relies on. |
| `entrypoint.sh` / `package.json` | Ensure Node runtime starts with proper environment (e.g., `NODE_ENV=production`) and that the `config` module loads the correct PostgreSQL credentials. |
| Other move scripts (e.g., Mongo processor) | May coordinate checkpoints by calling `getLatestLsn` and persisting the value to a state store. |

The module **does not** expose any CLI; it is purely a library used by the JavaScript processors.

---

## 6. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Leaked connections** – `pgConnect` is a global client; if `disconnectPg` is not called, the process may exhaust DB connections. | Service outage / DB throttling. | Enforce a try/finally pattern around the whole processing run; add a process‑exit handler that calls `disconnectPg`. |
| **Incorrect PK handling** – `mergeValues` builds a composite PK string; mismatched PK order between source and DB can cause duplicate or missing rows. | Data loss / duplication. | Validate PK order at start‑up; log the resolved PK list. |
| **Large CDC batches** – `getRowsByPageDelta` can return massive result sets if `limitSize` is omitted. | Memory pressure, OOM. | Always supply a sensible `limitSize` (e.g., 5000) and monitor row counts. |
| **SSL mis‑configuration** – SSL object may be empty; driver will attempt a non‑SSL connection when SSL is required. | Connection failures, security exposure. | Add explicit validation: if `pgsql.ssl` is defined but empty, abort with clear error. |
| **Global state race** – `processedLsn` / `ignoredLsn` are mutated elsewhere; concurrent runs could intermix LSN sets. | Incorrect log consumption, data gaps. | Serialize CDC processing per slot or protect arrays with a mutex / use a dedicated checkpoint store. |
| **Hard‑coded logical‑slot names** – Functions assume the caller supplies the correct slot; typo leads to empty reads. | Silent data loss. | Centralise slot names in config and validate existence on start‑up. |

---

## 7. Example: Running / Debugging the Module

```bash
# From the project root (assuming npm scripts are defined)
npm install   # ensure pg dependency is present
node -e "
  const pg = require('./move-Inventory/move-KYC-and-Esimshub/utils/pgSql');
  (async () => {
    try {
      const client = await pg.connectPg();
      // Example: fetch first 10 rows from source table
      const { rows, lastId } = await pg.getRowsByPage(
        { pk: 'id' },          // source definition (simplified)
        null,                 // no lastId -> start from beginning
        null,                 // db param unused
        'public', 'customer', // schema, table
        10, null
      );
      console.log('Rows:', rows);
      // Clean up
      await pg.disconnectPg();
    } catch (e) {
      console.error('Error:', e);
    }
  })();
"
```

*Debugging tips*

* Set `LOGGER_LEVEL=debug` (or adjust `config.logMsg` level) to see the generated SQL statements.
* Insert `console.log(pgConnect)` after `connectPg` to verify the client object.
* If `pgConnect` is undefined, check `globalDef.js` for the exported variable name and ensure it is required before calling any query function.

---

## 8. External Config / Environment Variables Referenced

| Config Path | Meaning |
|-------------|---------|
| `pgsql.multiUser` (bool) | Switches between per‑table credentials and a single credential set. |
| `pgsql.table` (string) | Table name used as a key when `multiUser` is true (determines which credential block to read). |
| `pgsql.<table>` / `pgsql.<table>Password` | Username / password for the specific table when `multiUser` is enabled. |
| `pgsql.user` / `pgsql.password` | Default credentials (single‑user mode). |
| `pgsql.host`, `pgsql.port`, `pgsql.database` | Connection endpoint. |
| `pgsql.ssl` (object) | Optional SSL options passed directly to the `pg` driver. |
| `pgsql.ssl` *must be non‑empty* to be applied; otherwise it is ignored. |

All of the above are accessed via `config.get(...)` – the underlying `config` module likely reads from a JSON/YAML file and/or environment variables.

---

## 9. Suggested TODO / Improvements

1. **Replace the global `pgConnect` with a returned client** – have each exported function accept the client (or use a class instance) to avoid hidden state and make the module thread‑safe.
2. **Enable proper connection pooling** – uncomment and finalize the `Pool` implementation, configure `max`, `idleTimeoutMillis`, and expose a `withClient(callback)` helper that automatically acquires/releases a client.

These changes will improve reliability, scalability, and testability of the move scripts.