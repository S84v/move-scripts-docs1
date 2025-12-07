**File:** `move-Inventory\move-MSPS\utils\utils.js`

---

## 1. High‑Level Summary
`utils.js` provides two lightweight, reusable helpers used throughout the MSPS move‑scripts:

1. **`sleep(ms)`** – Returns a Promise that resolves after the specified number of milliseconds, enabling async throttling or retry back‑off without blocking the event loop.  
2. **`toHHMMSS(num)`** – Converts a duration expressed in **milliseconds** (or sub‑millisecond fractions) into a human‑readable `HH:MM:SS.mmm` string, with special handling for negative values and sub‑second values (< 1 ms).

These functions are imported by the various execution‑queue, schema, and integration modules to pause processing loops and to log elapsed times in a consistent format.

---

## 2. Public API (Exported Functions)

| Export | Signature | Responsibility |
|--------|-----------|----------------|
| `sleep` | `sleep(ms: number): Promise<void>` | Creates a non‑blocking delay for *ms* milliseconds. |
| `toHHMMSS` | `toHHMMSS(num: number): string` | Formats a numeric duration (ms) into `HH:MM:SS.mmm`; returns `'unavailable'` for negative inputs, `'X nanosec'` for values < 1 ms. |

Both functions are pure (no external side‑effects) and synchronous except for the Promise returned by `sleep`.

---

## 3. Inputs, Outputs & Side Effects

| Function | Input | Output | Side Effects | Assumptions |
|----------|-------|--------|--------------|-------------|
| `sleep` | Positive integer (or float) **ms** | `Promise<void>` that resolves after *ms* | None (relies on `setTimeout`) | Node.js runtime, event‑loop remains responsive. |
| `toHHMMSS` | Numeric **num** (milliseconds, may be fractional) | Human‑readable string (`HH:MM:SS.mmm`, `'unavailable'`, or `'X nanosec'`) | None | Input is a finite number; negative values indicate unavailable data. |

No external services, files, or environment variables are accessed.

---

## 4. Interaction with Other Scripts

| Consumer | How It Uses `utils.js` |
|----------|------------------------|
| `move-MSPS\stack\msps\mspsExecutionQueue.js` (and similar queue scripts) | `await utils.sleep(delayMs)` to throttle polling of Oracle/Mongo queues. |
| `move-MSPS\stack\tableau\tableauExecutionQueue.js` | Same as above for rate‑limiting API calls. |
| Logging / reporting modules (e.g., any script that prints elapsed time) | `utils.toHHMMSS(durationMs)` to format runtime metrics for console or log files. |
| Test harnesses (if present) | May import `toHHMMSS` to verify timing output formatting. |

The module is required via standard Node `require` syntax, e.g.:

```js
const utils = require('../utils/utils');
```

Thus any script that needs a delay or a readable duration string will depend on this file.

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Excessive `sleep` values** – long delays can cause pipeline stalls. | Reduced throughput, missed SLAs. | Enforce upper‑bound checks in callers; centralize configuration of max back‑off. |
| **Incorrect input type** – passing non‑numeric or `NaN` to `sleep`/`toHHMMSS`. | Promise resolves immediately or `toHHMMSS` returns `NaN` strings. | Add defensive type‑checking (e.g., `Number.isFinite(ms)`). |
| **Clock drift / timer precision** – `setTimeout` is not real‑time accurate. | Minor timing inaccuracies in logs. | Acceptable for monitoring; if precise timing needed, switch to `process.hrtime`. |
| **Negative duration handling** – returns `'unavailable'` which may be mis‑interpreted. | Misleading logs. | Document expected sentinel value; callers should explicitly handle it. |
| **Missing module** – if `utils.js` is not packaged with a deployment. | Runtime `MODULE_NOT_FOUND` errors. | Include the file in the build artifact; add a CI check for required modules. |

---

## 6. Running / Debugging the Module

### Typical Usage (Operator)

```bash
# In a Node REPL or script
node -e "const u = require('./utils/utils'); 
(async () => { 
  console.log('Start'); 
  await u.sleep(2000); 
  console.log('After 2s'); 
  console.log(u.toHHMMSS(1234567)); // => 00:20:34.567 
})();"
```

### Debugging Steps

1. **Verify Export** – `console.log(require('./utils/utils'))` should show `{ sleep: [Function], toHHMMSS: [Function] }`.
2. **Check Timing** – Wrap `sleep` with `console.time`/`console.timeEnd` to confirm delay.
3. **Edge Cases** – Call `toHHMMSS(-5)`, `toHHMMSS(0.5)`, `toHHMMSS(999)` and verify output matches expectations.
4. **Unit Tests** – If a test suite exists, run `npm test` (or equivalent) to ensure no regression.

---

## 7. External Configuration / Environment Dependencies

- **None**. The module is self‑contained; it does not read environment variables, config files, or external services.

---

## 8. Suggested Improvements (TODO)

1. **Input Validation** – Guard against non‑numeric or out‑of‑range arguments and throw a descriptive error rather than silently returning unexpected strings.
2. **Add Unit Tests** – Provide a minimal test suite (e.g., using Jest or Mocha) covering typical, boundary, and error cases for both functions to catch regressions during future refactors.