**File:** `move-Inventory\move-BOSS-Maximity\utils\utils.js`

---

## 1. High‑level Summary
`utils.js` provides two generic helper utilities used throughout the BOSS‑Maximity move pipeline:

1. **`sleep(ms)`** – an async delay returning a Promise that resolves after *ms* milliseconds.  
2. **`toHHMMSS(num)`** – converts a duration expressed in **milliseconds** (or sub‑millisecond fractions) into a human‑readable `HH:MM:SS.mmm` string, with special handling for negative or sub‑millisecond values.

Both functions are pure (no side‑effects) and are exported for consumption by the various processor modules (`fullSync`, `deltaSync`, etc.) and other utility files (`mongo.js`, `mssql.js`, etc.).

---

## 2. Important Functions & Responsibilities

| Export | Signature | Responsibility |
|--------|-----------|----------------|
| `sleep` | `async function sleep(ms: number): Promise<void>` | Returns a Promise that resolves after *ms* milliseconds using `setTimeout`. Used to throttle loops, wait for external resources, or implement retry back‑off. |
| `toHHMMSS` | `function toHHMMSS(num: number): string` | Formats a numeric duration (ms) into `HH:MM:SS.mmm`. Handles: <0 → `'unavailable'`; <1 → nanosecond representation; otherwise normal HH:MM:SS.mmm. |

*No classes are defined in this file.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Function | Input | Output | Side Effects | Assumptions |
|----------|-------|--------|--------------|-------------|
| `sleep` | `ms` – non‑negative integer (milliseconds) | `Promise<void>` resolved after delay | Uses Node’s `setTimeout`; no I/O | Caller runs in a Node environment that supports Promises. |
| `toHHMMSS` | `num` – numeric duration (ms, may be fractional) | Human‑readable string (`HH:MM:SS.mmm`, `'unavailable'`, or `'X nanosec'`) | Pure function – no side effects | Input is a finite number; negative values indicate unavailable timing. |

---

## 4. External Configuration / Environment Dependencies

- **None**. The module relies only on built‑in Node APIs (`setTimeout`, `Promise`).  
- It is imported via `require('../utils/utils')` (or similar) by other scripts; no environment variables or external files are needed.

---

## 5. Integration Points (How This File Connects to the Rest of the System)

| Consumer | Typical Use‑Case |
|----------|------------------|
| `processor/v1/fullSync.js`, `processor/v3/deltaSync.js`, etc. | `await sleep(1000)` to pause between API calls or DB batch inserts. |
| Logging / reporting modules (e.g., `processor/main.js`) | `toHHMMSS(durationMs)` to display elapsed time in console or audit logs. |
| Test harnesses | Used to simulate latency or format timing metrics. |

Because the file is exported as an object `{sleep, toHHMMSS}`, any script that does `const {sleep, toHHMMSS} = require('../utils/utils');` can use the helpers.

---

## 6. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Unbounded `sleep` values** – accidental large delay (e.g., `sleep(1e9)`) can stall the job. | Production pipeline hangs, SLA breach. | Validate `ms` before calling (e.g., cap at configurable max). |
| **Incorrect time unit assumptions** – callers may pass seconds instead of ms, leading to huge formatted strings. | Misleading logs, difficulty troubleshooting. | Document expected unit; add a runtime check (`if (num > 86400000) warn...`). |
| **Blocking the event loop** – `sleep` is async, but misuse with `while (!condition) await sleep(0)` can create busy‑wait loops. | CPU waste, reduced throughput. | Prefer exponential back‑off and limit retry count. |
| **`toHHMMSS` overflow** – very large numbers may produce hour values >99, breaking fixed‑width log parsers. | Log parsing errors. | Truncate or switch to days format for >99h durations. |

---

## 7. Running / Debugging Example

```bash
# From the project root (assuming Node >=12)
node -e "
  const {sleep, toHHMMSS} = require('./move-Inventory/move-BOSS-Maximity/utils/utils');
  (async () => {
    console.log('Start:', toHHMMSS(0));
    await sleep(1500);
    console.log('After 1.5s:', toHHMMSS(1500));
  })();
"
```

**Debugging tips**

1. **Step‑through** – Use `node --inspect-brk` and attach Chrome DevTools; set breakpoints inside `utils.js` to verify conversion logic.
2. **Unit test** – Add a quick test file:
   ```js
   const assert = require('assert');
   const {toHHMMSS} = require('./utils');
   assert.strictEqual(toHHMMSS(12345), '00:00:12.345');
   ```
   Run with `npm test` (or `node test.js`) to catch regressions.

---

## 8. Suggested TODO / Improvements

1. **Input validation** – Guard `sleep(ms)` against non‑numeric, negative, or excessively large values and throw a descriptive error.
2. **Configurable time‑unit handling** – Extend `toHHMMSS` to accept an optional `unit` parameter (`'ms' | 's' | 'ns'`) so callers can pass seconds directly without pre‑conversion.

---