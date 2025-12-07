**File:** `move-Inventory\move-KYC-and-Esimshub\utils\globalDef.js`

---

## 1. High‑Level Summary
`globalDef.js` establishes a collection of **application‑wide constant strings** and **utility getters** that are attached to the Node.js `global` object. These constants represent process states, operation types, data‑type identifiers, and change‑capture actions used throughout the KYC‑and‑Esimshub move scripts (e.g., `processor/main.js`, `processor/deltaSync/*.js`, `mongo_processor.js`). By loading this file early (typically via `require` in the entrypoint), every module can reference the same symbolic values without importing a dedicated constants module.

---

## 2. Important Constants & Their Responsibilities  

| Constant | Category | Meaning / Usage |
|----------|----------|-----------------|
| `INITIALIZING`, `RUNNING`, `ABORTED`, `SLEEPING`, `PAUSED`, `SYNCHRONIZING`, `STOPPED`, `FINALIZING`, `FAILED`, `FINISHED` | **Process State** | Lifecycle flags for a job (e.g., stored in MongoDB status field). |
| `RUN`, `INIT`, `DELTA`, `FULL` | **Job Mode** | Indicates whether a job is a full sync, delta sync, init, or a run‑once execution. |
| `TEXT`, `STRING`, `BOOLEAN`, `DATE`, `NUMBER`, `NULL`, `TRUE`, `FALSE` | **Data Type Tokens** | Used by transformation logic to map source types to target schema. |
| `UPDATE`, `DELETE`, `INSERT` | **SQL‑style DML** | Generic CRUD operation identifiers. |
| `WALINSERT`, `WALDELETE`, `WALUPDATE` | **WAL (Write‑Ahead Log) Actions** | Specific to the `wal2json` delta‑sync processor. |
| `DELTASYNC` | **Sync Type** | Marker for delta‑sync jobs. |
| `JOB`, `AFTER` | **Metadata Keys** | Keys used in job definition objects. |
| `BULKOPSEXECMS`, `TOTALINJECTMSFORFULLSYNC`, `DBWRITES` | **Performance Metrics** | Metric names stored in job logs / monitoring. |
| `TMP` | **Temp Namespace** | Prefix for temporary collections or files. |
| `COLLECTIONREPLACED`, `COLLECTIONLEFTINTACT` | **Collection Status** | Log messages for collection handling. |
| `INVALIDDATE`, `NOCASEFOUND`, `NOLSNFOUND` | **Error / Info Tokens** | Standardised messages for validation failures. |

---

## 3. Utility Getters  

| Getter | Description |
|--------|-------------|
| `global.__stack` | Returns the current call‑stack as an array of `CallSite` objects (via temporary override of `Error.prepareStackTrace`). |
| `global.__line` | Returns the line number of the caller (`__stack[1]`). |
| `global.__function` | Returns the name of the caller function (`__stack[1]`). |

These helpers are used for **debug logging** in other modules (e.g., `processor/*.js`) to embed source location in log entries.

---

## 4. Inputs, Outputs, Side Effects & Assumptions  

| Aspect | Detail |
|--------|--------|
| **Inputs** | None – the file only defines globals. |
| **Outputs** | Populates the Node.js `global` namespace with the constants and getters above. |
| **Side Effects** | Global namespace pollution; any later `require` of this file overwrites the same keys. |
| **Assumptions** | - The runtime environment is a single‑process Node.js instance (no worker threads that would share a different global). <br>- All other scripts load this file **before** they reference any of the defined globals. <br>- No other library in the codebase defines conflicting global keys. |

---

## 5. Connection to Other Scripts & Components  

| Connected File / Component | How It Uses `globalDef.js` |
|----------------------------|----------------------------|
| `entrypoint.sh` → `node processor/main.js` | `main.js` does `require('../utils/globalDef')` early to bootstrap constants. |
| `processor/main.js` | Reads `global.RUNNING`, `global.FULL`, etc., to drive job state machine. |
| `processor/fullSync.js` & `processor/deltaSync/*.js` | Use `global.UPDATE/INSERT/DELETE` and WAL constants to translate change events. |
| `processor/mongo_processor.js` | Writes status fields (`global.FINISHED`, `global.FAILED`) into MongoDB job documents. |
| Logging utilities (e.g., custom logger) | Use `global.__line` and `global.__function` for enriched log messages. |
| Azure Pipelines / Deployment YAML | No direct reference, but the pipeline ensures the file is packaged and available in the container image. |

Because the constants are global, **any future module that `require`s this file automatically gains access** without needing explicit imports.

---

## 6. Potential Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Global namespace collisions** – another library may define the same key, causing silent overwrites. | Unexpected behavior, hard‑to‑trace bugs. | Scope constants to a dedicated namespace (e.g., `global.KYC = { ... }`) or export via `module.exports`. |
| **Accidental mutation** – code could reassign a global constant. | State corruption, downstream modules misinterpret values. | Freeze the object (`Object.freeze(global)`) after definition or use `const` within a module export. |
| **Performance overhead of stack getters** – `__stack` creates a new `Error` each call. | Minor CPU cost when used in high‑frequency loops. | Limit use to debug level; guard with environment flag (`process.env.DEBUG`). |
| **Testing isolation** – globals persist across test cases, causing cross‑test leakage. | Flaky tests. | Reset or re‑initialize globals in test setup/teardown, or refactor to a non‑global module. |
| **Lack of Type Safety** – all values are plain strings; typos are not caught at compile time. | Runtime errors. | Adopt TypeScript or JSDoc enums for compile‑time checking. |

---

## 7. Running / Debugging the File  

*The file itself does not execute any logic.* To verify it loads correctly:

```bash
# From the project root (container or local dev)
node -e "require('./move-Inventory/move-KYC-and-Esimshub/utils/globalDef'); console.log(global.RUNNING, global.UPDATE);"
```

Expected output (example):

```
processing UPDATE
```

When debugging a consumer module (e.g., `processor/fullSync.js`), you can inspect the source location helpers:

```js
console.log(`Entered ${global.__function} at line ${global.__line}`);
```

If the log shows `undefined` for `__function` or `__line`, ensure `globalDef.js` is required **before** the logger is instantiated.

---

## 8. External Config / Environment Variables  

`globalDef.js` does **not** read any external configuration, environment variables, or files. Its sole purpose is to define in‑process constants.

---

## 9. Suggested Improvements (TODO)

1. **Refactor to a module export** – replace the `global` assignments with a plain object exported via `module.exports`. Consumers would then `const G = require('../utils/globalDef');` improving testability and avoiding global pollution.
2. **Add JSDoc / TypeScript definitions** – document each constant’s intended use and provide enum types to catch misspellings at compile time.

---