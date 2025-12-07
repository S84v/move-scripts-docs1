**File:** `move-Inventory\move-MSPS\utils\globalDef.js`

---

## 1. High‑Level Summary
`globalDef.js` injects three read‑only global getters into the Node.js runtime:

* `__stack` – returns the current call‑stack as an array of `CallSite` objects.  
* `__line`  – returns the source‑line number of the caller (the second frame in the stack).  
* `__function` – returns the name of the calling function (second frame).

These helpers are used throughout the Move‑MSPS codebase for lightweight debugging and log‑enrichment without pulling in a full‑featured logger. By loading this module once (typically at the very start of a script), any subsequent file can reference `__line`, `__function`, or `__stack` directly.

---

## 2. Key Definitions

| Symbol | Type | Responsibility |
|--------|------|----------------|
| `global.__stack` | getter | Captures a fresh `Error` object, temporarily overrides `Error.prepareStackTrace` to expose the raw stack array, then restores the original formatter. Returns the full stack trace (array of `CallSite`). |
| `global.__line` | getter | Reads `__stack[1]` (the caller’s frame) and returns its line number via `CallSite.getLineNumber()`. |
| `global.__function` | getter | Reads `__stack[1]` and returns the caller’s function name via `CallSite.getFunctionName()`. |

*No classes or exported functions are defined; the module’s side‑effect is the registration of the three globals.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | None – the getters are invoked at runtime by the consuming code. |
| **Outputs** | Values returned by the getters (`Array<CallSite>`, `number`, `string | null`). |
| **Side‑Effects** | • Adds three properties to the Node.js `global` object.<br>• Temporarily mutates `Error.prepareStackTrace` each time a getter runs (restored immediately). |
| **Assumptions** | • Running under Node.js (V8 engine) where `Error.prepareStackTrace` is supported.<br>• No other module redefines the same globals (name collision risk).<br>• The code is loaded **before** any use of the globals. |
| **External Dependencies** | None (standard library only). |

---

## 4. Integration with the Rest of the Move‑MSPS System

| Connection Point | How it is used |
|------------------|----------------|
| **All execution‑queue scripts** (`*_ExecutionQueue.js`) | Typically `require('../../utils/globalDef')` at the top; later they embed `__line` / `__function` in log statements (e.g., `logger.info(\`[${__function}:${__line}] Starting job\`)`). |
| **Schema / Mongo modules** (`*_MongoSchema.js`) | May use `__function` for error context when constructing Mongoose models. |
| **Custom utilities** (`utils/*.js`) | Any utility that needs to report its origin without passing explicit metadata can read `__line`/`__function`. |
| **Testing harnesses** | Test scripts may import this file to verify that stack‑trace helpers work as expected. |

Because the module mutates the global namespace, it must be required **once** (usually by a bootstrap file such as `index.js` or the first script in a pipeline). Subsequent `require` calls are no‑ops due to Node’s module cache.

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Global namespace pollution** – other libraries may also define `__line` or `__function`. | Unexpected overrides, hard‑to‑track bugs. | Prefix globals (e.g., `__msps_line`) or move to a scoped helper object (`const dbg = require('./globalDef')`). |
| **Performance overhead** – each getter creates an `Error` and walks the stack. | Increased CPU usage in high‑throughput loops. | Use the helpers only in error paths or debug logging; avoid in tight loops. |
| **Interference with custom `Error.prepareStackTrace`** – if another module permanently changes this function, the temporary override may produce incorrect results. | Corrupted stack data, misleading logs. | Detect existing overrides and restore them after each getter, or document that this module must be loaded before any custom formatter. |
| **Missing source maps** – in transpiled code (e.g., Babel/TypeScript) line numbers may point to generated files. | Debugging confusion. | Ensure source‑map support is enabled or limit usage to native JS files. |
| **Potential memory leak** – the getter captures a stack array that could be retained inadvertently. | Gradual memory growth. | Verify that callers do not store `__stack` long‑term; encourage use of `__line`/`__function` only. |

---

## 6. Running / Debugging the Module

1. **Load the helper** – Add at the very top of any script:  
   ```js
   require('./utils/globalDef');   // path relative to the script
   ```
2. **Use in code** – Example snippet:  
   ```js
   function processRecord(rec) {
       logger.debug(`[${__function}:${__line}] processing ${rec.id}`);
       // … business logic …
   }
   ```
3. **Verify** – Run a simple script:  
   ```js
   require('./utils/globalDef');
   console.log('Line:', __line);          // should print the line number of this console.log
   console.log('Function:', __function); // null (top‑level)
   console.log('Stack depth:', __stack.length);
   ```
4. **Debugging** – If `__line` returns `undefined` or an unexpected number, check:  
   * The file is being required **before** the call site.  
   * No other module has overwritten `Error.prepareStackTrace` permanently.  
   * Node version supports `CallSite` methods (≥ v6).  

---

## 7. External Configuration / Environment Variables

`globalDef.js` does **not** read any configuration files, environment variables, or command‑line arguments. Its behavior is entirely deterministic based on the runtime environment.

---

## 8. Suggested Improvements (TODO)

1. **Encapsulate in a non‑global utility** – Export a function `getCallerInfo()` that returns `{ line, function, stack }`. This eliminates global pollution and makes the dependency explicit.  
2. **Add TypeScript declaration** – Provide a `global.d.ts` (or module export) so IDEs can type‑check uses of `__line`/`__function` and avoid accidental misuse.

---