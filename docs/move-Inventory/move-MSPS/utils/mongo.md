**File:** `move-Inventory\move-MSPS\utils\mongo.js`

---

### 1. High‑Level Summary
`mongo.js` is a shared utility module that abstracts MongoDB connectivity and common write‑path operations for the Move‑MSPS data‑move platform. It provides functions to (a) open/close a Mongo client based on the current operational mode, (b) generate sequential identifiers stored in a `counters` collection, (c) execute bulk write batches while aggregating statistics and handling write errors, and (d) persist runtime statistics to a dedicated aggregation database. All other Move‑MSPS scripts that interact with MongoDB (e.g., schema definitions, execution‑queue handlers) import this module.

---

### 2. Public API – Key Functions & Responsibilities  

| Exported Symbol | Responsibility | Important Details |
|-----------------|----------------|-------------------|
| **`connect(url)`** | Returns a `Promise` that resolves to a native MongoDB client **only** when the configuration indicates Mongo is required (`opMode.source` or `opMode.target` equals `"mongo"`). | Logs connection attempts; resolves with `undefined` when Mongo is not needed. |
| **`disconnect(conn)`** | Gracefully closes a Mongo client if it exists; otherwise logs a no‑op. | Returns a `Promise`. |
| **`getSequence(db, seqId)`** | Atomically increments a counter document in the `counters` collection and returns a formatted sequence string (`<prefix><paddedNumber>`). Creates the counter document on‑first use. | Uses `padLeft` helper; logs errors; returns `undefined` on failure. |
| **`bulkWrite(bulkOpsArr, summ)`** | Executes one or more bulk operation batches (`BulkWriteOperation` objects) against a collection, aggregates write statistics (`totals`) and timing (`chronos`), and captures any write errors. | - Accepts an array of bulk operation objects (each created via `collection.initializeOrderedBulkOp()` or `initializeUnorderedBulkOp()`).<br>- Returns `{ hasErrors, bulkResultArr }`.<br>- Populates `summ.totals` with counts for inserted, modified, upserted, removed, and write errors.<br>- Logs detailed warnings on failures. |
| **`logDbWriteErrors(bulkWriteResult, summDocs)`** | Scans the `bulkResultArr` returned by `bulkWrite` and logs each individual write error via `logDbErrorDoc`. Returns a boolean indicating if any errors were found. |
| **`logDbErrorDoc(err, doc, summDocs, logLvl, sourceName, functionName)`** | Formats a MongoDB error, writes it to the logger, and increments per‑error‑code counters in `summDocs.dbErrors`. |
| **`updateStats(stats)`** | Persists a statistics object to the aggregation database (`mongo.dbAggr` → `mongo.statsCollection`) using an upsert on a fixed `_id` (`seq`). | Uses `config.getConstructed('mongoDb')` to obtain a pre‑connected client. |

*Helper*: `padLeft(nr, n, str)` – left‑pads a number with a given character (defaults to `'0'`).

---

### 3. Inputs, Outputs & Side Effects  

| Function | Primary Inputs | Primary Outputs | Side Effects |
|----------|----------------|-----------------|--------------|
| `connect` | Mongo connection string (`url`) | `Promise<MongoClient|undefined>` | Opens a TCP connection to Mongo; logs debug messages. |
| `disconnect` | `MongoClient` instance (or `null`) | `Promise<void>` | Closes the socket; logs if no client. |
| `getSequence` | `db` (MongoDB database handle), `seqId` (string) | `Promise<string>` (e.g., `ORD000000123`) | Reads & updates `counters` collection; may insert a new counter document; logs errors. |
| `bulkWrite` | `bulkOpsArr` (array of bulk operation objects), `summ` (object with `totals` & `chronos`) | `Promise<{hasErrors:boolean, bulkResultArr:Array}>` | Executes writes; updates `summ.totals` & `summ.chronos`; logs warnings/errors. |
| `logDbWriteErrors` | Result from `bulkWrite`, `summDocs` (stats container) | `boolean` (error presence) | Calls `logDbErrorDoc` for each write error; updates `summDocs.dbErrors`. |
| `logDbErrorDoc` | Mongo error object, offending document, stats container, optional log level & source info | `void` | Writes to logger; increments per‑error counters. |
| `updateStats` | `stats` (plain object) | `Promise<void>` | Upserts into `mongo.statsCollection`; logs on failure. |

**Assumptions / External Dependencies**

* **Configuration (`config/config`)** – Provides:
  * `opMode.source` / `opMode.target` – determines if Mongo is needed.
  * `mongoDb` – a constructed Mongo client (used by `updateStats`).
  * `mongo.dbAggr`, `mongo.statsCollection` – aggregation DB name & collection.
  * `seq` – identifier for the stats document.
* **Logger (`config.logMsg`)** – Custom logger exposing methods: `debug`, `info`, `warning`, `error`, `unexpectedErr`, `unusual`, etc.
* **MongoDB driver** – `mongodb` npm package, version supporting `MongoClient.connect` with `{useNewUrlParser:true}`.
* **Calling scripts** – All Move‑MSPS modules that need DB access import this utility (e.g., `mspsMongoSchema.js`, `tableauMongoSchema.js`, execution‑queue scripts).

---

### 4. Integration Points – How This Module Connects to the Rest of the System  

| Consumer | How It Uses `mongo.js` |
|----------|------------------------|
| **Schema definition files** (`*_MongoSchema.js`) | Import `connect` to obtain a client, then use the client to create collections, indexes, or perform initial data loads. |
| **Execution‑queue handlers** (`*_ExecutionQueue.js`) | Use `bulkWrite` to persist batches of transformed records into target collections; use `logDbWriteErrors` to surface duplicate‑key or validation errors. |
| **Statistics collector** (any script that reports run metrics) | Calls `updateStats` after a job finishes to store aggregated counters (e.g., total writes, errors). |
| **Sequence generator** (e.g., order‑id creation) | Calls `getSequence` to obtain a unique, formatted identifier for each new entity. |
| **Orchestration scripts** (top‑level `move-*.js`) | Use `connect` at start‑up, pass the client to downstream modules, and invoke `disconnect` on graceful shutdown. |

Because the module resolves the client lazily based on `opMode`, a single process can run in *source‑only*, *target‑only*, or *dual* mode without unnecessary connections.

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Unbounded bulk batch size** – Very large `bulkOpsArr` can exhaust memory or hit Mongo’s 100 MB batch limit. | Job failure, OOM, partial writes. | Enforce a configurable max batch size (e.g., 10 k ops) and split larger arrays before calling `bulkWrite`. |
| **Silent no‑op when Mongo is disabled** – `connect` resolves with `undefined`; downstream code may assume a client exists and throw `TypeError`. | Unexpected crashes in source‑only runs. | Add defensive checks in callers (`if (!client) return;`) or make `connect` reject when a client is required but not configured. |
| **Sequence race conditions** – Multiple processes increment the same counter concurrently; `findOneAndUpdate` with `$inc` is atomic, but missing indexes on `_id` could degrade performance. | Duplicate IDs or performance bottlenecks. | Ensure an index on `counters._id` (default primary key) and monitor the `counters` collection for lock contention. |
| **Error handling gaps** – `bulkWrite` catches errors but may swallow the original exception (`hasErrors` flag only). | Lost context for troubleshooting. | Propagate the original error object (or at least its stack) to the caller after logging. |
| **Connection leakage** – If a script exits without calling `disconnect`, the client may stay open. | Resource exhaustion on the DB server. | Register a process `exit`/`SIGINT` handler that calls `disconnect` on any open client. |
| **Hard‑coded collection name `'counters'`** – If a different collection is required, code must be changed. | Inflexibility across environments. | Parameterise the counters collection name via config. |

---

### 6. Running / Debugging the Module  

**Typical usage pattern (async/await):**

```javascript
// exampleScript.js
const mongoUtil = require('./utils/mongo');
const config = require('config/config');

(async () => {
  try {
    const client = await mongoUtil.connect(config.getConstructed('mongoDbUrl'));
    if (!client) {
      console.log('Mongo not required for this run.');
      return;
    }

    const db = client.db(config.get('mongo.dbTarget'));

    // Example: generate a new order ID
    const orderId = await mongoUtil.getSequence(db, 'order');
    console.log('Generated Order ID:', orderId);

    // Example: bulk insert
    const bulk = db.collection('orders').initializeOrderedBulkOp();
    bulk.insert({ _id: orderId, status: 'NEW' });
    // ... add more ops ...

    const stats = { totals: {}, chronos: {} };
    const bulkResult = await mongoUtil.bulkWrite([bulk], stats);

    if (mongoUtil.logDbWriteErrors(bulkResult, stats.totals)) {
      console.warn('Some writes failed – see logs for details.');
    }

    // Persist stats
    await mongoUtil.updateStats(stats.totals);
  } catch (e) {
    console.error('Fatal error:', e);
  } finally {
    await mongoUtil.disconnect(client);
  }
})();
```

**Debugging tips**

* Set `config.logMsg` level to `debug` to see connection attempts and each bulk batch execution.
* Inspect `stats.totals` after `bulkWrite` – it contains `dbWrites`, `dbInserted`, `dbModified`, `dbRemoved`, `dbUpserted`, and `writeErrors`.
* If `hasErrors` is true, run `logDbWriteErrors` to get per‑document error details; the logger will include the offending document JSON.
* Use MongoDB’s `profile` or `currentOp` to verify that bulk operations are arriving as expected.

---

### 7. External Configuration & Environment Variables  

| Config Path | Meaning | Used By |
|-------------|---------|----------|
| `opMode.source` / `opMode.target` | `"mongo"` indicates that the script must open a Mongo client for source or target data. | `connect` |
| `mongoDb` (constructed) | Pre‑instantiated `MongoClient` (or connection string) used by `updateStats`. | `updateStats` |
| `mongo.dbAggr` | Name of the aggregation database where stats are stored. | `updateStats` |
| `mongo.statsCollection` | Collection name for the stats document. | `updateStats` |
| `seq` | Fixed `_id` value for the stats document (e.g., `"runStats"`). | `updateStats` |
| `mongoDbUrl` (or similar) | Full Mongo connection URI passed to `connect`. | `connect` (caller supplies) |
| Logger configuration (`logMsg`) | Controls log levels and output destinations. | All functions |

If any of these keys are missing, the module will either log a debug message (Mongo disabled) or throw an error when attempting to use the missing value.

---

### 8. Suggested Improvements (TODO)

1. **Batch‑size Guard & Automatic Splitting** – Add a helper `splitBulkOps(bulkOpsArr, maxOps = 10000)` that returns an array of bulk batches respecting MongoDB’s 100 MB / 10 k‑op limits, and integrate it into `bulkWrite`.
2. **Retry Wrapper for Transient Errors** – Implement a generic `retryAsync(fn, attempts = 3, backoff = 200)` that can be used around `bulkOp.execute()` to automatically retry on network timeouts or write‑concern errors, reducing manual error‑handling code.

---