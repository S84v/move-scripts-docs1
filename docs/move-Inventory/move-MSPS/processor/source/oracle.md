**File:** `move-Inventory\move-MSPS\processor\source\oracle.js`  

---

## 1. High‑level Summary
`oracle.js` implements the **Oracle source connector** for the MSPS “move” pipeline. It reads a list of extraction stages from the `oracleQueue` configuration, connects to an Oracle database, executes each stage’s SQL (optionally in paged mode), and streams the result rows into MongoDB collections using bulk write operations. The module also maintains per‑stage execution statistics in a MongoDB “stats” document, supports temporary collection swaps for zero‑downtime refreshes, and periodically reconnects to Oracle to avoid long‑running session time‑outs.

---

## 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`exec()`** (exported) | Orchestrates the whole Oracle queue: creates the Oracle connection, iterates over each stage, fetches rows (paged or full), calls `processRows()` to load into MongoDB, updates execution stats, and finally disconnects. |
| **`recreateConnection()`** | Disconnects any existing Oracle session and opens a fresh one using credentials from `config.get('oracle')`. |
| **`getRowsByPage(stage)`** | Implements paging for large result sets: builds a ranked sub‑query, loops over page windows, fetches rows, concatenates them, and reconnects after every *percentageReconnect* % of rows (default 25 %). |
| **`buildQuery(query, pageOrderBy)`** | Helper that rewrites a raw `SELECT … FROM …` into a ranked sub‑query suitable for paging. |
| **`processRows(stage, rows)`** | Transforms raw Oracle rows into MongoDB write operations according to the stage’s schema (`mongoSchema.js`). Handles optional temporary collection creation, index building, bulk write execution, and final collection swap/drop logic. |
| **`getTempColName(stage)`** | Generates a unique temporary collection name using the process PID. |
| **`mongo.getOp(stage, row, replacements, bulkContainer)`** (called) | Returns a MongoDB write operation (insert/update) for a single row based on the stage’s mapping rules. |
| **`mongoUtils.bulkWrite([...], summary)`** (called) | Executes a bulk write against MongoDB and aggregates timing/record‑count info into `summary`. |
| **`mongoUtils.updateStats(stats)`** (called) | Persists the in‑memory `stats` object to the MongoDB “stats” collection. |
| **`mongoUtils.logDbWriteErrors(...)`** (called) | Logs any write errors returned from a bulk operation. |

---

## 3. Inputs, Outputs & Side‑effects  

| Aspect | Details |
|--------|---------|
| **Inputs** | • `config` – runtime configuration (Oracle credentials, MongoDB connection, `opMode`, etc.)<br>• `queue` – `oracleQueue` array of stage objects (name, query, pageMode, pageSize, pageOrderBy, target collection, etc.)<br>• Oracle DB – accessed via `utils/oracle` (expects `connect`, `execute`, `disconnect`). |
| **Outputs** | • MongoDB collections populated per stage (either the target collection or a temporary one that may be renamed).<br>• Execution statistics stored in a MongoDB “stats” document (`executionQueue` entries, elapsed time, record counts, error objects, summary of bulk loads). |
| **Side‑effects** | • Network connections to Oracle and MongoDB.<br>• Potential creation/dropping of temporary collections and indexes.<br>• Logging to the configured logger (debug/info/warn/error). |
| **Assumptions** | • Oracle queries are syntactically correct and return a column set where the last column is the generated `my_rank` (removed after paging).<br>• The `mongoSchema` for each stage defines `replacements`, optional `options.dropCollection`, `options.lossPerc`, and `options.indexes`.<br>• `opMode.target` is either `'mongo'` or `'fileRepo'` (the latter is not implemented here).<br>• The MongoDB user has privileges to create/drop collections, create indexes, and rename collections. |

---

## 4. Integration Points  

| Component | How `oracle.js` Connects |
|-----------|--------------------------|
| **`config/executionQueue.js`** | Exposes `oracleQueue` which drives the per‑stage loop in `exec()`. |
| **`processor/target/mongo.js`** | Provides `getOp()` that builds the actual MongoDB write operation for each row. |
| **`utils/oracle.js`** | Supplies low‑level Oracle connection handling (`connect`, `execute`, `disconnect`). |
| **`utils/mongo.js`** | Supplies bulk write helpers (`bulkWrite`, `logDbWriteErrors`). |
| **`config/mongoSchema.js`** | Supplies per‑stage mapping rules and collection options used in `processRows()`. |
| **`config/config.js`** | Central logger (`logMsg`) and generic `config.get()` / `config.getConstructed()` used throughout. |
| **`main.js` (processor entry point)** | Likely imports `oracle.js` and invokes `exec()` as part of the overall pipeline (e.g., after source selection). |
| **`entrypoint.sh`** | Shell wrapper that runs the Node process (`node processor/main.js`), sets environment variables, and captures exit codes. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Long‑running Oracle session timeout** – the script reconnects only every 25 % of rows. | Query failures, partial data loads. | Make `percentageReconnect` configurable; monitor Oracle session idle timeout; optionally enable Oracle “keep‑alive” or increase timeout. |
| **Bulk write memory pressure** – accumulating rows in a single bulk container before flushing. | Out‑of‑memory crashes on very large stages. | Reduce the bulk flush threshold (currently 5 000 ops) or make it configurable; add back‑pressure handling. |
| **Accidental data loss on collection swap** – if `lossPerc` is mis‑configured, a new collection with fewer rows could replace the production collection. | Permanent loss of records. | Enforce a minimum acceptable `lossPerc` (e.g., –5 %); add a pre‑swap validation step that compares row counts and aborts on unexpected loss. |
| **Index creation race conditions** – indexes are built on the temporary collection while data is being bulk‑written. | Index build may block writes or cause timeouts. | Use `background: true` (already set) and optionally create indexes *after* bulk load completes. |
| **Uncaught promise rejections** – many async calls have only generic `catch` blocks; errors may be swallowed. | Silent failures, incomplete stats. | Propagate errors up to `exec()` and fail the stage; add explicit error handling and alerting. |
| **Hard‑coded column removal (`r.slice(0,-1)`)** – assumes the last column is always the rank. | Incorrect data if query shape changes. | Derive column positions from metadata or make the rank column name configurable. |

---

## 6. Running / Debugging the Module  

1. **Typical execution** (via the pipeline entry point):  
   ```bash
   # From the project root
   npm install   # ensure dependencies are present
   ./entrypoint.sh   # wrapper that runs node processor/main.js
   ```
   `entrypoint.sh` sets `NODE_ENV` (development/production) which selects the appropriate config file (`config/development.json` or `config/production.json`).

2. **Direct invocation for testing** (e.g., from a REPL or unit test):  
   ```bash
   node -e "require('./processor/source/oracle').exec().catch(console.error)"
   ```

3. **Debugging tips**  
   - Set `LOG_LEVEL=debug` (or adjust logger config) to see per‑stage row counts and bulk‑write progress.  
   - Use `NODE_OPTIONS=--inspect` to attach a debugger (Chrome DevTools or VS Code).  
   - Insert `console.log` or `logger.debug` statements before/after `oraConn.execute` to verify the generated paged query.  
   - Check the MongoDB “stats” collection to confirm that `executionQueue` entries are being updated.  
   - If the script hangs on a large stage, monitor Oracle session activity (`v$session`) and MongoDB memory usage.

---

## 7. External Configuration & Environment Variables  

| Config File / Variable | Purpose |
|------------------------|---------|
| `config/config.js` (via `config.get`) | Provides logger, generic config accessor, and constructed objects (`mongoDb`, `stats`). |
| `config/executionQueue.js` (`oracleQueue`) | Defines the list of stages to run (name, query, pageMode, pageSize, pageOrderBy, target collection, etc.). |
| `config/mongoSchema.js` (`schema[stage]`) | Supplies field mapping (`replacements`), collection options (`dropCollection`, `lossPerc`), and index definitions for each stage. |
| `config/development.json` / `config/production.json` | Holds environment‑specific values: Oracle connection string (`oracle.host`, `oracle.user`, `oracle.password`), MongoDB connection URI (`mongo.uri`), `opMode.target`, `percentageReconnect` (if overridden), logging levels, etc. |
| Environment variables (e.g., `NODE_ENV`, `LOG_LEVEL`) | Control which config file is loaded and the verbosity of logging. |
| `process.pid` (used in `getTempColName`) | Guarantees a unique temporary collection name per process instance. |

If any of the above keys are missing, the script will throw at runtime (e.g., `config.get('oracle')`).

---

## 8. Suggested TODO / Improvements  

1. **Make paging parameters configurable** – expose `percentageReconnect`, `bulkFlushSize`, and the rank column name via the `oracleQueue` or a dedicated config section to avoid hard‑coded values.  
2. **Add robust validation before collection swap** – implement a pre‑swap checksum (e.g., hash of a sample of rows) and enforce a stricter loss‑percentage policy, aborting the rename if the new collection deviates beyond an acceptable threshold.  

---