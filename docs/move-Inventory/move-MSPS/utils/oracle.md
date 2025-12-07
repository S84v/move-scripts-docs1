**File:** `move-Inventory\move-MSPS\utils\oracle.js`

---

### 1. High‑Level Summary
`oracle.js` is a thin wrapper around the Node oracledb driver that provides two async helpers – `connect` and `disconnect` – used throughout the Move‑MSPS stack to obtain and release Oracle database connections. The module activates only when the global configuration (`opMode.source` or `opMode.target`) declares Oracle as a source or target system, otherwise it returns `undefined` and logs a no‑op message. All Oracle‑dependent scripts import this utility to centralise connection handling and logging.

---

### 2. Exported Functions & Responsibilities
| Export | Type | Responsibility |
|--------|------|-----------------|
| `connect(attributes)` | `async function` | If Oracle is required by the current run‑mode, logs a debug message, attempts `oracledb.getConnection(attributes)`, logs any error, re‑throws, and returns the live connection object. If Oracle is not required, logs a debug notice and returns `undefined`. |
| `disconnect(conn)` | `async function` | Safely closes a supplied Oracle connection (`conn.close()`). If `conn` is falsy, logs a debug notice that no client was instantiated. |

*No classes are defined; the module exports a plain object with the two functions.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Element | Details |
|---------|---------|
| **Input – `attributes`** | Object passed to `oracledb.getConnection`. Typically contains `user`, `password`, `connectString`, and optional pool settings. Caller must source these from secure vaults or environment variables. |
| **Output – `connect`** | Returns a live `oracledb.Connection` instance when Oracle is required; otherwise returns `undefined`. |
| **Output – `disconnect`** | Returns a resolved promise after `conn.close()` completes; no value is returned. |
| **Side Effects** | • Writes debug / error messages via the shared `logger` (configured in `config/config`).<br>• Opens a network socket to the Oracle listener.<br>• May allocate resources in the Oracle client library (e.g., OCI handles). |
| **Assumptions** | • `config` module is present and exposes `logMsg` (a Winston‑style logger) and `opMode` keys.<br>• The `oracledb` driver is installed and compatible with the Oracle server version.<br>• Caller handles credential protection; the utility does **not** mask passwords in logs. |

---

### 4. Integration Points  

| Component | How it uses `oracle.js` |
|-----------|------------------------|
| **Data‑move scripts** (e.g., `msps2ExecutionQueue.js`, `tableauExecutionQueue.js`) | Import `oracle.js` to obtain a connection when reading from or writing to Oracle tables. |
| **Global configuration (`config/config.js`)** | Supplies `opMode.source` / `opMode.target` flags and may also contain default connection attributes that callers forward to `connect`. |
| **Other utils (`mongo.js`, `globalDef.js`)** | Operate in parallel but do **not** depend on Oracle; they share the same logger instance, ensuring consistent log formatting across all data sources. |
| **Orchestration layer / Scheduler** | Calls the script entry‑points that eventually invoke `oracle.connect()`; the returned connection is passed down the call chain. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Leaked Oracle connections** (e.g., missing `disconnect` in error paths) | Enforce `try … finally` around every `connect` usage; consider adding a wrapper that auto‑closes on promise rejection. |
| **Incorrect `opMode` configuration** leading to silent no‑ops (connection never created) | Validate `opMode` at application start; fail fast if required source/target is missing. |
| **Credential exposure in logs** (error objects may contain passwords) | Scrub sensitive fields before logging or use `logger.error({msg: e.message, stack: e.stack})` without the raw error object. |
| **Driver version mismatch** causing runtime errors | Pin `oracledb` version in `package.json` and test against the target Oracle release during CI. |
| **Unbounded concurrent connections** (multiple scripts may call `connect` simultaneously) | Introduce a connection pool (see TODO) or limit parallelism via a semaphore in the orchestration layer. |

---

### 6. Example Usage (Operator / Developer)

```bash
# From a terminal, run a Move‑MSPS job that uses Oracle
node scripts/runMspsJob.js --source=oracle --target=mongo
```

**Debugging steps inside a script:**

```javascript
const oracle = require('../utils/oracle');
const logger = require('config/config').logMsg;

async function example() {
  const attrs = {
    user: process.env.ORACLE_USER,
    password: process.env.ORACLE_PWD,
    connectString: process.env.ORACLE_CONNECT
  };

  let conn;
  try {
    conn = await oracle.connect(attrs);
    if (!conn) {
      logger.warn('Oracle not required for this run‑mode');
      return;
    }
    const result = await conn.execute('SELECT COUNT(*) FROM some_table');
    logger.info('Row count:', result.rows[0][0]);
  } catch (err) {
    logger.error('Oracle operation failed', err);
  } finally {
    await oracle.disconnect(conn);
  }
}
example();
```

*Set environment variables (`ORACLE_USER`, `ORACLE_PWD`, `ORACLE_CONNECT`) before invoking the script. The logger will emit debug lines indicating whether a connection was attempted.*

---

### 7. External Configuration & Files Referenced  

| Reference | Purpose |
|-----------|---------|
| `config/config` (module) | Provides `logMsg` (logger) and `opMode` flags (`source`, `target`). May also expose default Oracle connection attributes used by callers. |
| Environment variables (commonly used) | `ORACLE_USER`, `ORACLE_PWD`, `ORACLE_CONNECT` – not read directly here but expected by callers that build the `attributes` object. |
| `oracledb` NPM package | Underlying driver; must be installed and compatible with the Oracle client libraries on the host. |

---

### 8. Suggested Improvements (TODO)

1. **Introduce a shared connection pool** – replace per‑call `getConnection` with `oracledb.createPool` at process start and expose `getConnectionFromPool`/`releaseConnection` helpers to limit concurrent sessions and improve performance.
2. **Add explicit input validation** – verify that required fields (`user`, `password`, `connectString`) are present and non‑empty before invoking `oracledb.getConnection`; throw a descriptive error early to aid troubleshooting.