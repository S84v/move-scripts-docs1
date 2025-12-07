**File:** `move-Inventory\move-BOSS-Maximity\utils\mssql.js`

---

## 1. High‑Level Summary
`mssql.js` is a thin wrapper around the **node‑mssql** driver that provides two async helpers – `connect(attributes)` and `disconnect()` – for establishing and tearing down a Microsoft SQL Server connection used throughout the BOSS‑Maximity move scripts. It pulls a logger from the shared configuration module (`config/config`) and abstracts the driver’s `ConnectionPool` vs. legacy `Connection` selection, returning a ready‑to‑use connection object to callers (e.g., the processor modules that read/write inventory data).

---

## 2. Exported API & Responsibilities  

| Export | Type | Responsibility |
|--------|------|-----------------|
| `connect(attributes)` | `async function` | Creates a new MSSQL connection (pool if available) using the supplied connection attributes, logs the attempt, and returns the live connection object. |
| `disconnect()` | `async function` | Gracefully closes the previously created connection (if any) and logs a debug message when no connection exists. |

*Internal state*: a module‑scoped variable `conn` holds the active connection instance between calls.

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Element | Details |
|---------|---------|
| **Input – `attributes`** | Plain object containing MSSQL connection parameters (`user`, `password`, `server`, `database`, optional `options` such as `encrypt`). Expected to be supplied by callers (usually `config.mssql` or similar). |
| **Output – `connect`** | Resolves to the instantiated `ConnectionPool` or `Connection` object (the driver’s connection handle). Callers can invoke `.request()` etc. |
| **Output – `disconnect`** | Resolves to `undefined`. Side effect is closing the underlying TCP socket / pool. |
| **Side Effects** | - Network I/O to the SQL Server.<br>- Logging via `config.logMsg` (debug & error levels). |
| **Assumptions** | - `config/config` exports a `logMsg` object with `debug`/`error` methods (e.g., Winston/Bunyan).<br>- The `mssql` package is installed and compatible with the runtime Node version.<br>- Caller will handle any thrown errors (e.g., connection failure).<br>- Only one concurrent connection per process (module‑scoped `conn`). |

---

## 4. Interaction with Other Components  

| Component | How it uses `mssql.js` |
|-----------|------------------------|
| `processor/v*/deltaSync.js` & `processor/v*/fullSync.js` | Import `{ connect, disconnect }` to open a SQL connection, run SELECT/INSERT/UPDATE statements against the BOSS/Maximity inventory tables, then close the connection. |
| `utils/globalDef.js` (or other utils) | May reference the same logger instance; `mssql.js` re‑uses that logger, ensuring consistent log formatting across the move suite. |
| `config/config.js` | Supplies the logger and possibly the MSSQL connection attributes (e.g., `config.mssql`). The wrapper does **not** read the attributes itself; callers pass them in. |
| `move-Inventory\move-BOSS-Maximity\processor\main.js` | Orchestrates the overall run; typically calls `connect` at the start of a job and `disconnect` in a `finally` block. |

*Note*: Because the module holds a single `conn`, concurrent jobs in the same Node process will share the same connection object. If the broader system runs multiple parallel jobs, they must coordinate or each job should use its own process.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Connection leak** – `disconnect()` not called (e.g., exception before finally) | Exhausts DB connections, job stalls | Enforce `try … finally` around every `connect` usage; consider adding a process‑wide `process.on('exit')` hook that forces `disconnect()`. |
| **Unbounded pool growth** – using `ConnectionPool` without limits | DB resource exhaustion | Pass explicit pool options (`pool.max`, `pool.min`, `pool.idleTimeoutMillis`) via `attributes`. |
| **Missing or malformed logger** – `config.logMsg` undefined | Silent failures, no audit trail | Validate logger presence at module load; fallback to console logger if absent. |
| **Version mismatch** – driver API changes (e.g., removal of `Connection`) | Runtime errors on startup | Pin `mssql` version in `package.json`; add unit test that checks `sql.ConnectionPool` existence. |
| **Credential exposure** – attributes may be read from env without sanitization | Security breach | Ensure credentials are sourced from a vault or encrypted env vars; never log the full attribute object. |

---

## 6. Running / Debugging Example  

```bash
# From the project root (assuming npm scripts are defined)
npm run start   # or node processor/main.js
```

**Programmatic debugging** (inside a processor module):

```javascript
const { connect, disconnect } = require('../utils/mssql');
const cfg = require('config/config');

async function testConnection() {
  try {
    const conn = await connect(cfg.mssql);   // cfg.mssql holds the attribute object
    const result = await conn.request()
                             .query('SELECT TOP 1 * FROM dbo.Inventory');
    console.log('Test row:', result.recordset[0]);
  } catch (err) {
    console.error('Connection test failed:', err);
  } finally {
    await disconnect();   // always close
  }
}

testConnection();
```

*Debug tip*: Increase logger level to `debug` in the configuration to see the “Connecting to mssql Database” message and any error stack traces.

---

## 7. External Configuration / Environment Variables  

| Reference | Purpose |
|-----------|---------|
| `config/config` (imported as `config`) | Provides `logMsg` (logger) and typically also the MSSQL connection attributes (`config.mssql`). |
| Environment variables (indirect) | The `config` module may read DB credentials from `DB_USER`, `DB_PASSWORD`, `DB_SERVER`, etc. Verify the `config` implementation to know which env vars are required. |
| No direct file system or network resources are referenced beyond the SQL Server endpoint defined in the attributes. |

---

## 8. Suggested Improvements (TODO)

1. **Add Connection Pool Reuse** – Store the created `ConnectionPool` in a module‑level cache and return the same instance on subsequent `connect` calls (with reference counting) to avoid repeatedly opening/closing pools in high‑frequency jobs.

2. **Implement Retry Logic** – Wrap the `conn.connect()` call with exponential back‑off (e.g., using `async-retry`) to improve resilience against transient network glitches.

---