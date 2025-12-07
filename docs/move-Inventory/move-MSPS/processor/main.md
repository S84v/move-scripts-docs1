**File:** `move-Inventory\move-MSPS\processor\main.js`

---

## 1. High‑Level Summary
`main.js` is the orchestration entry point for the MSPS inventory move. It reads the runtime configuration (`opMode`) to determine the source system, logs the selected source‑to‑target direction, and dispatches execution to the appropriate source‑specific processor (currently only Oracle). All other processing steps are delegated to that source module.

---

## 2. Important Classes / Functions

| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `exec` | `async function` (exported) | Reads `opMode` from the central config, logs the chosen source/target, selects the correct source driver (`processor/source/oracle`) and returns its promise. Errors on unknown sources. |
| `config` | Imported module (`config/config`) | Provides configuration access (`config.get`) and a pre‑configured logger (`config.logMsg`). |
| `logger` | `object` (from `config.logMsg`) | Centralised logging (info, error). |
| `oracle` | Imported module (`processor/source/oracle`) | Expected to expose an `exec()` that performs the Oracle‑specific extraction, transformation, and load steps. |

*No classes are defined in this file; it only exports a single function.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - `config` object (populated from `config/*.json` via the `config` package). <br> - `opMode` property containing `{ source: string, target: string }`. |
| **Outputs** | - Returns the promise from the selected source driver (`oracle.exec()`). <br> - On error, returns a rejected promise with a descriptive string. |
| **Side Effects** | - Writes an informational log line with the source/target mapping. <br> - Writes an error log line if the source is unsupported. |
| **Assumptions** | - `config` module is correctly initialised before this file is required. <br> - `opMode.source` will be a string matching a known driver (currently only `'oracle'`). <br> - The `oracle` module implements an `exec` returning a promise that resolves when the move completes. <br> - No direct I/O (DB, file, network) occurs here; all such actions are performed downstream. |

---

## 4. Connection to Other Scripts & Components

| Component | Relationship |
|-----------|--------------|
| `config/config.js` | Supplies the `config` object and the `logMsg` logger used here. |
| `processor/source/oracle/index.js` (or similar) | The concrete implementation invoked when `opMode.source === 'oracle'`. It likely contains the Oracle DB connection, query logic, and downstream calls to target adapters. |
| `app.js` (root entry point) | Imports `processor/main.js` and calls `exec()` as part of the overall job pipeline. |
| `executionQueue.js` (in `config/`) | May schedule or throttle calls to `main.exec()`; not referenced directly here but part of the broader orchestration. |
| Target adapters (e.g., SFTP, REST, Mongo) | Invoked downstream from the source driver; `main.js` only selects the source. |
| Environment variables (e.g., `NODE_ENV`) | Indirectly affect which config file (`development.json` vs `production.json`) is loaded by the `config` package. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Invalid or missing `opMode` configuration** | Job aborts early, no data moved. | Validate `opMode` at startup; add a sanity‑check that both `source` and `target` are non‑empty strings. |
| **Unsupported source value** | Rejection with generic error; may cause unhandled promise if caller does not catch. | Extend the `default` case to list supported sources; ensure callers always `await` and `catch`. |
| **Failure to load `oracle` module** (e.g., missing file, syntax error) | Runtime exception before any logging. | Wrap the `require` in a try/catch and log a clear error; add unit tests that verify module resolution. |
| **Uncaught promise rejection** (if caller forgets to handle) | Process may exit with non‑zero code, losing visibility. | Enforce a top‑level error handler in `app.js` that logs and exits gracefully. |
| **Configuration drift between environments** (different `opMode` values) | Unexpected source/target mapping in production. | Store `opMode` in a version‑controlled config file; use CI checks to compare dev vs prod values. |

---

## 6. Example Execution / Debugging Workflow

1. **Run the full move (as an operator)**  
   ```bash
   # Assuming Docker or direct node execution
   cd move-Inventory/move-MSPS
   npm install   # ensures dependencies
   NODE_ENV=production node app.js   # app.js will call processor/main.exec()
   ```

2. **Run only this processor for quick sanity check**  
   ```bash
   node -e "require('./processor/main').exec()
       .then(()=>console.log('Done'))
       .catch(err=>console.error('Failed:', err))"
   ```

3. **Debug with breakpoints**  
   - Start Node with the inspector:  
     ```bash
     node --inspect-brk processor/main.js
     ```  
   - Attach Chrome DevTools or VS Code; set a breakpoint on the `switch` statement to verify `opMode` values.

4. **Inspect configuration**  
   ```bash
   node -e "console.log(require('config/config').get('opMode'))"
   ```

5. **Unit test (if test framework present)**  
   ```bash
   npm test   # test.js likely contains a stub for main.exec()
   ```

---

## 7. External Configuration / Environment Dependencies

| Item | Usage |
|------|-------|
| `config/config.js` (via `config` npm package) | Loads JSON files (`development.json`, `production.json`) based on `NODE_ENV`. Provides `config.get('opMode')` and `config.logMsg`. |
| `opMode` key in config files | Determines which source driver to invoke (`source`) and where data should be sent (`target`). |
| `NODE_ENV` (environment variable) | Controls which config file is loaded; must be set to `development` or `production` as appropriate. |
| Any secrets (DB credentials, API keys) referenced inside the source driver (`oracle`) | Not used directly here but required downstream; ensure they are present in the environment or secret store. |

---

## 8. Suggested TODO / Improvements

1. **Add explicit validation of `opMode`** – before the `switch`, verify that `source` and `target` are defined and belong to an allowed set; throw a descriptive error otherwise.  
2. **Implement a fallback or plugin architecture** – replace the hard‑coded `switch` with a dynamic driver registry (`require('processor/source/' + opMode.source)`) that can gracefully handle missing modules and list available sources.  

---