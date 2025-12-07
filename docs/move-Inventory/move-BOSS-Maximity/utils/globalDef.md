**File:** `move-Inventory\move-BOSS-Maximity\utils\globalDef.js`

---

## 1. High‑Level Summary
`globalDef.js` establishes a set of process‑state constants on the Node.js `global` object (e.g., `INITIALIZING`, `RUNNING`, `FINISHED`) and injects three debugging helpers (`__stack`, `__line`, `__function`). These globals are consumed throughout the move‑scripts to report the current lifecycle stage of a job and to provide concise source‑location information for log statements and error handling.

---

## 2. Important Globals / Functions

| Global / Property | Type | Responsibility |
|-------------------|------|----------------|
| `global.INITIALIZING` | `string` | Marks the start‑up phase of a move job. |
| `global.RUNNING` | `string` | Indicates active processing of records. |
| `global.ABORTED` | `string` | Set when a job is terminated early (e.g., user abort). |
| `global.SLEEPING` | `string` | Used when the job is idle awaiting a schedule or external trigger. |
| `global.PAUSED` | `string` | Represents a manual pause state. |
| `global.SYNCHRONIZING` | `string` | Specific to delta/full sync phases. |
| `global.STOPPED` | `string` | Final stopped state after graceful shutdown. |
| `global.FINALIZING` | `string` | Cleanup phase before termination. |
| `global.FAILED` | `string` | Set on unrecoverable error. |
| `global.FINISHED` | `string` | Normal successful completion. |
| `global.__stack` | *getter* | Returns the current call‑stack as an array of `CallSite` objects (via `Error.prepareStackTrace`). |
| `global.__line` | *getter* | Returns the source line number of the caller (`__stack[1]`). |
| `global.__function` | *getter* | Returns the name of the caller function (`__stack[1]`). |

*No exported functions or classes – the file’s side‑effect is the augmentation of `global`.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Detail |
|--------|--------|
| **Inputs** | None (the module does not read files, env vars, or arguments). |
| **Outputs** | None returned; side‑effects are the added properties on `global`. |
| **Side Effects** | • Pollutes the global namespace (intended for intra‑process sharing).<br>• Alters `Error.prepareStackTrace` temporarily to capture a stack trace. |
| **Assumptions** | • Running under Node.js (v10+).<br>• No other module overwrites the same global keys.<br>• Callers are tolerant of the synchronous stack capture overhead. |

---

## 4. Integration with Other Scripts

| Consumer | Typical Usage |
|----------|---------------|
| `processor/**/*.js` (e.g., `fullSync.js`, `deltaSync.js`) | Set `global.state = global.RUNNING` (or other constants) to drive status reporting and to decide flow control (e.g., abort if `global.state === global.ABORTED`). |
| Logging utilities (likely in `processor/main.js` or a custom logger) | Use `global.__line` and `global.__function` to enrich log entries: `logger.info(\`[${global.__function}:${global.__line}] processing record\`)`. |
| Monitoring / health‑check endpoints | Export the current `global.state` to external dashboards (e.g., Prometheus metrics). |
| Error handling blocks | Throw an error with contextual info: `new Error(\`${global.__function} failed at line ${global.__line}\`)`. |

Because the constants live on `global`, any module that `require`s this file (directly or indirectly via the entry point) can read/write them without explicit import.

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Global namespace collision** – another library may define the same property names. | Unexpected state overrides, hard‑to‑trace bugs. | Prefix constants (e.g., `MOVE_STATE_INITIALIZING`) or encapsulate in a dedicated module exported via `module.exports`. |
| **Performance overhead** – each access to `__stack` creates an `Error` object and walks the stack. | Increased CPU usage in tight loops or high‑throughput processing. | Use `__stack` only in error paths or debug logging; avoid in per‑record loops. |
| **Security / information leakage** – stack traces may expose internal file paths. | Potential exposure in logs sent to external systems. | Mask or redact file paths before shipping logs; restrict debug logging to internal environments. |
| **Unintended mutation** – any script can change the state constants at runtime. | Inconsistent job status, race conditions. | Treat the constants as read‑only (Object.freeze) and expose a separate mutable `global.jobState` variable for runtime changes. |
| **Node version incompatibility** – reliance on `Error.prepareStackTrace` may break in future Node releases. | Runtime errors or missing stack info. | Add a fallback that returns `null` if the API is unavailable; monitor Node upgrade notes. |

---

## 6. Example: Running / Debugging the File

1. **Load the module** (normally done by the main entry point):  
   ```js
   // entrypoint.js
   require('./utils/globalDef');   // populates globals
   const processor = require('./processor/main');
   processor.start();              // processor will read/write global state
   ```

2. **Inspect state from a REPL** (useful for ad‑hoc debugging):  
   ```bash
   $ node
   > require('./utils/globalDef');
   > global.RUNNING
   'processing'
   > global.__function   // returns undefined because we are at top‑level
   undefined
   > (function test(){ console.log(global.__function, global.__line); })()
   test 2
   ```

3. **Debug a failing sync** – add a temporary log in `deltaSync.js`:  
   ```js
   logger.error(`[${global.__function}:${global.__line}] Delta sync error: ${err.message}`);
   ```

   When the error occurs, the log will contain the exact function name and line number.

---

## 7. External Configuration / Environment Variables

`globalDef.js` does **not** read any external configuration, environment variables, or other files. Its behavior is deterministic based solely on the Node runtime.

---

## 8. Suggested Improvements (TODO)

1. **Encapsulate constants in a module** – replace direct `global` mutation with an exported immutable object, e.g.,  
   ```js
   const STATES = Object.freeze({
     INITIALIZING: 'initializing',
     RUNNING: 'processing',
     // …
   });
   module.exports = STATES;
   ```
   Consumers would `const STATES = require('../utils/globalDef');` and reference `STATES.RUNNING`. This eliminates accidental global pollution and improves testability.

2. **Add a lightweight stack helper** – provide a function `getCallerInfo()` that returns `{functionName, lineNumber}` without constructing a full `Error` each call, or cache the result when debugging is disabled via an env flag (`DEBUG_STACK=false`). This reduces overhead in production loops.