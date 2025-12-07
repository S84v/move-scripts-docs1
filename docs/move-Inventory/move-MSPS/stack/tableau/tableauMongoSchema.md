**File:** `move-Inventory\move-MSPS\stack\tableau\tableauMongoSchema.js`

---

## 1. High‑level Summary
This module defines the MongoDB collection schema for the *Tableau user‑mapping* data set used in the MSPS inventory move stack. It exports a plain JavaScript object (`schema`) that describes the primary key, collection options (including a forced drop on load and a loss‑percentage tolerance), index definitions, and the list of fields with their data types. The schema is consumed by generic schema‑initialisation utilities (e.g., the execution‑queue loader) to create or rebuild the `tableauUserMapping` collection before data‑move jobs run.

---

## 2. Key Export(s) and Their Responsibilities
| Export | Type | Responsibility |
|--------|------|-----------------|
| `schema` | `object` | Holds the full definition of the `tableauUserMapping` collection: <br>• `_id` composite key (`userId`, `secsId`, `buId`). <br>• `options` controlling collection creation (`dropCollection`, `lossPerc`, `indexes`). <br>• `fields` array describing each column (`userId`, `secsId`, `buId`, `userType`). |

*No functions or classes are defined in this file; it is a pure data definition.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | - The schema object is imported by other scripts (e.g., `tableauExecutionQueue.js`). <br>- Implicit runtime configuration such as MongoDB connection URI, database name, and collection name are supplied by the surrounding framework (usually via environment variables or a central config module). |
| **Outputs** | - The exported `schema` object is passed to a generic MongoDB schema loader which creates the collection, applies indexes, and optionally drops the existing collection. |
| **Side Effects** | - `options.dropCollection: true` instructs the loader to **drop the entire `tableauUserMapping` collection** before recreating it. <br>- `lossPerc: 50` is a tolerance flag used by the loader to decide whether a data‑loss event (e.g., missing fields) is acceptable; the exact semantics are defined elsewhere. |
| **Assumptions** | - A MongoDB driver/ORM wrapper exists that understands the schema format used here. <br>- The execution environment has sufficient privileges to drop and recreate collections. <br>- The downstream data‑move jobs will repopulate the collection after it is recreated. |

---

## 4. Interaction with Other Scripts & Components

| Component | Connection Point | Role in the Flow |
|-----------|------------------|------------------|
| `tableauExecutionQueue.js` | `require('./tableauMongoSchema')` → uses `schema` to initialise the collection before queuing jobs. | Ensures the collection exists with the correct indexes and options before any Tableau‑related move tasks are scheduled. |
| `tableau.json` | Provides higher‑level configuration (e.g., source/target system identifiers) that the execution queue reads; indirectly depends on the collection being present. | Drives the overall move orchestration; the schema file guarantees the data store matches the expected shape. |
| Generic *schema loader* (e.g., `loadMongoSchema.js` in the shared library) | Consumes the exported `schema` object to perform `db.createCollection`, `collection.createIndex`, and conditional `dropCollection`. | Centralised logic that translates the declarative schema into MongoDB commands. |
| Environment / Config module | Supplies `MONGO_URI`, `MONGO_DB_NAME`, possibly a flag to disable `dropCollection` in production. | Provides connection details and runtime toggles that affect how the schema is applied. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Uncontrolled collection drop** (`dropCollection: true`) | Complete loss of existing `tableauUserMapping` data if the loader runs unintentionally in production. | - Guard the loader with an environment flag (e.g., `ALLOW_TABLEAU_DROP`). <br>- Require a manual approval step or a separate “re‑create” script. |
| **High loss tolerance** (`lossPerc: 50`) | Up‑stream validation may silently accept up to 50 % missing/invalid rows, leading to data quality issues. | - Review the business rule; lower the threshold or make it configurable per environment. |
| **Missing index enforcement** | If the loader fails to create the unique composite index, duplicate rows could be inserted later. | - Add post‑creation verification (e.g., `listIndexes` check) and abort on mismatch. |
| **Schema drift** | Adding/removing fields without updating dependent transformation scripts can cause runtime errors. | - Maintain a versioned schema registry and enforce CI checks that any schema change triggers a review of downstream jobs. |

---

## 6. Running / Debugging the Schema

1. **Typical execution path** (triggered by the orchestration framework):  
   ```bash
   node scripts/loadMongoSchema.js --schema ./stack/tableau/tableauMongoSchema.js
   ```
   The loader reads the exported `schema`, connects to MongoDB (using `process.env.MONGO_URI`), drops the collection if allowed, creates it, and applies the defined indexes.

2. **Manual verification** (useful for debugging):  
   ```js
   // debugSchema.js
   const { MongoClient } = require('mongodb');
   const { schema } = require('./stack/tableau/tableauMongoSchema');

   (async () => {
     const client = await MongoClient.connect(process.env.MONGO_URI);
     const db = client.db(process.env.MONGO_DB_NAME);
     const coll = db.collection('tableauUserMapping');

     // Show current indexes
     console.log('Existing indexes:', await coll.indexes());

     // Optionally drop & recreate (only in dev)
     if (process.argv.includes('--recreate')) {
       await coll.drop().catch(() => {}); // ignore if not exists
       await db.createCollection('tableauUserMapping');
       await coll.createIndex(
         { "_id.userId": 1, "_id.secsId": 1, "_id.buId": 1 },
         { name: 'pk', unique: true }
       );
       console.log('Recreated collection with index.');
     }

     await client.close();
   })();
   ```
   Run with `node debugSchema.js --recreate` in a non‑production environment to confirm the index definition and collection options.

3. **Logging**: The generic loader should emit logs such as `Dropping collection tableauUserMapping`, `Creating index pk`, and `Schema load completed`. Verify these messages in the orchestration logs.

---

## 7. External Configuration & Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| `process.env.MONGO_URI` | MongoDB connection string (host, port, auth). |
| `process.env.MONGO_DB_NAME` | Target database where `tableauUserMapping` lives. |
| `process.env.ALLOW_TABLEAU_DROP` (suggested) | Boolean flag to enable/disable the `dropCollection` behaviour in production. |
| Central **config module** (e.g., `config.js`) | May map logical collection names to physical names, provide retry policies, and expose the `lossPerc` threshold to other components. |

The schema file itself does **not** read any env vars; it relies on the loader to interpret `options` and apply them using the above configuration.

---

## 8. Suggested Improvements (TODO)

1. **Make `dropCollection` configurable per environment** – replace the hard‑coded `true` with a runtime flag (e.g., read from `process.env.ALLOW_TABLEAU_DROP`) to prevent accidental data loss in production.
2. **Add JSON‑Schema validation** – augment the `fields` definition with validation rules (e.g., required, pattern) and integrate them into the loader so malformed documents are rejected early.

---