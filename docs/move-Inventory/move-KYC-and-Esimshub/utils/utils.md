**File:** `move-Inventory\move-KYC-and-Esimshub\utils\utils.js`

---

## 1. High‑Level Summary
`utils.js` is a tiny, pure‑JavaScript helper module that provides two asynchronous‑friendly utilities used throughout the KYC‑and‑Esimshub move‑scripts:

1. **`sleep(ms)`** – returns a Promise that resolves after the specified number of milliseconds, enabling non‑blocking pauses in async workflows (e.g., rate‑limiting API calls, waiting for external processes).  
2. **`toHHMMSS(num)`** – converts a duration expressed in **milliseconds** (or sub‑millisecond fractions) into a human‑readable `HH:MM:SS.mmm` string, falling back to “nanosec” for sub‑millisecond values and “unavailable” for negative inputs.

Both functions are stateless, have no side‑effects, and are imported by the various processor modules (`fullSync.js`, `deltaSync/*.js`, etc.) for logging and pacing purposes.

---

## 2. Exported Functions & Responsibilities

| Export | Signature | Responsibility | Return |
|--------|-----------|----------------|--------|
| `sleep` | `sleep(ms: number): Promise<void>` | Pause execution for *ms* milliseconds without blocking the event loop. | A Promise that resolves after the timeout. |
| `toHHMMSS` | `toHHMMSS(num: number): string` | Convert a numeric duration (ms) to a formatted time string. Handles negative, sub‑millisecond, and large values. | Formatted string (`HH:MM:SS.mmm`, `nanosec`, or `unavailable`). |

*Implementation notes*  
- `sleep` uses `setTimeout` wrapped in a Promise – safe for async/await.  
- `toHHMMSS` truncates fractional seconds, pads components to two digits, and pads milliseconds to three digits.  

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Function | Input | Expected Type / Range | Output | Side‑Effects | Assumptions |
|----------|-------|-----------------------|--------|--------------|-------------|
| `sleep` | `ms` | Non‑negative integer (ms). | `Promise<void>` | None (no I/O, no state mutation). | Caller handles the returned Promise (e.g., `await sleep(1000)`). |
| `toHHMMSS` | `num` | Numeric duration in **milliseconds**. May be < 1 (fractional ms) or negative. | Human‑readable string. | None. | Input is a raw duration from timers, DB latency measurements, etc. |

No external services, files, databases, queues, SFTP, or APIs are touched.

---

## 4. Connection to Other Scripts & Components

| Consumer | Typical Use‑Case |
|----------|------------------|
| `processor/fullSync.js` | `await sleep(5000)` to throttle bulk API calls; `toHHMMSS(elapsedMs)` for logging batch duration. |
| `processor/deltaSync/text.js` & `wal2json.js` | Insert short pauses when polling PostgreSQL logical replication slots; format latency metrics for monitoring dashboards. |
| `utils/mongo.js` / `utils/pgSql.js` | May use `sleep` when retrying transient DB connection failures; `toHHMMSS` for reporting query execution times. |
| `processor/main.js` | Central orchestration script that logs overall run time using `toHHMMSS`. |

Because the module is **pure**, it can be required anywhere in the codebase without affecting dependency graphs or runtime order.

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Unbounded `sleep` durations** – a mis‑configured value (e.g., reading a large number from env) could stall the pipeline for minutes/hours. | Production bottleneck, SLA breach. | Validate any external source of the `ms` argument before calling `sleep`; enforce a maximum (e.g., 30 000 ms). |
| **Incorrect time unit assumptions** – callers may mistakenly pass seconds instead of milliseconds, leading to overly long pauses. | Unexpected delays, resource waste. | Document the required unit clearly; add a wrapper (`sleepSec`) if seconds are a common input. |
| **`toHHMMSS` overflow** – very large numbers (> Number.MAX_SAFE_INTEGER) could lose precision. | Mis‑reported metrics, confusing logs. | Guard against values > 2^53‑1; truncate or cap before formatting. |
| **Missing async handling** – calling `sleep` without `await` may lead to fire‑and‑forget behavior. | Race conditions, incomplete data moves. | Encourage lint rule (`no-floating-promise`) or code review checklist. |

---

## 6. Example: Running / Debugging the Module

```bash
# From any script in the move‑KYC-and-Esimshub tree
node -e "
  const { sleep, toHHMMSS } = require('./utils/utils');
  (async () => {
    console.log('Start:', new Date());
    await sleep(1500);
    console.log('Elapsed:', toHHMMSS(1500));
  })();
"
```

**Debugging tips**

| Situation | Approach |
|-----------|----------|
| Unexpected long pause | Insert `console.log('sleep ms:', ms, new Date())` before calling `sleep`. |
| Wrong time format | Log the raw `num` passed to `toHHMMSS` and verify unit (ms). |
| Promise never resolves | Verify that `ms` is a finite number; `setTimeout` will ignore `NaN` and never fire. |

Because the functions are synchronous (except for the Promise wrapper), standard Node.js debugging (`node --inspect`) works without special instrumentation.

---

## 7. External Configuration / Environment Variables

`utils.js` **does not** read any configuration files, environment variables, or external resources. All inputs are supplied by the caller. If future requirements introduce configurable defaults (e.g., default sleep duration), they should be injected via parameters or a dedicated config module (e.g., `config/config.js`).

---

## 8. Suggested Improvements (TODO)

1. **Input Validation Layer** – add a small helper (`validateMs`) that throws a descriptive error for non‑numeric, negative, or excessively large values before invoking `setTimeout`.  
2. **Unit Tests** – introduce a Jest/Mocha test suite covering edge cases: negative numbers, sub‑millisecond values, large durations, and Promise resolution timing (using fake timers).  

These changes will increase reliability and give the team confidence when modifying or extending the utilities.