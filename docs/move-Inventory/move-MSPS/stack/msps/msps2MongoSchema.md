**File:** `move-Inventory\move-MSPS\stack\msps\msps2MongoSchema.js`  

---

## 1. High‑level Summary
This module defines the **MongoDB collection schema** for the “sims” entity used by the *MSPS‑2* data‑move pipeline. It specifies collection‑level options (drop‑on‑load, loss tolerance, indexes), the primary key (`iccid`), field types, and per‑field transformation directives that are interpreted by the generic Mongo target driver (`processor/target/mongo.js`). The schema drives how rows extracted from the Oracle source are materialised as documents in the target Mongo database.

---

## 2. Exported Object & Responsibilities  

| Export | Type | Responsibility |
|--------|------|-----------------|
| `schema` | `object` | Holds a single top‑level collection definition (`sims`). All downstream components import this object to know how to create the collection, map source columns, and apply field‑level operations. |

### 2.1 Collection‑level options (`schema.sims.options`)
| Option | Meaning | Production impact |
|--------|---------|-------------------|
| `dropCollection: true` | The collection is dropped and recreated each run. | Guarantees a clean slate but can cause total data loss if the run aborts after drop. |
| `lossPerc: 0.1` | Acceptable percentage of rows that may be dropped (e.g., due to validation failures). | Used by the target driver to decide whether to abort the job. |
| `indexes` | Array of index specifications. Currently creates a single ascending index on `sponsors.imsi` named `sponsor`. | Improves query performance for downstream services that filter on sponsor IMSI. |

### 2.2 Primary key (`_id`)
`_id: ['iccid']` – the `iccid` field becomes the Mongo document `_id`. Guarantees uniqueness and enables upserts.

### 2.3 Field definitions
Each entry in `fields` describes:

* **name** – source column / target document field.
* **type** – primitive (`int`, `string`, `date`) or complex (`array`, `object`).
* **operation** *(optional)* – transformation directive interpreted by the Mongo driver.

Key operation types observed:

| Operation type | Purpose |
|----------------|---------|
| `ignore` | Field is read from source but **not persisted** in the target document. |
| `createImsiArray` | Builds the `sponsors` array from the various `*Sponsor` / `*Imsi` columns. |
| `createBizUnitsArray` | Constructs `bizUnits` array from business‑unit related columns (not shown in this file but defined elsewhere). |
| `mergeArray` (fields) | Merges the listed source fields (`msisdn1`, `msisdn2`) into a single `msisdns` array. |
| `createCustObject` | Packs a set of customer‑related columns into a nested `cust` object. |

All other fields are copied verbatim, respecting the declared `type`.

---

## 3. Inputs, Outputs, Side‑effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Input source** | Rows produced by `processor/source/oracle.js` (Oracle query result set). Column names must match the `name` entries above. |
| **Output target** | MongoDB collection `sims` in the database defined by the *production* config (`move-MSPS/config/production.json`). |
| **Side‑effects** | • Dropping the collection on each run (potential full data loss). <br>• Creation of the `sponsor` index. |
| **Assumptions** | • Mongo connection string, credentials, and database name are supplied via the global configuration (`production.json`) or environment variables (`MONGO_URI`, `MONGO_DB`). <br>• The generic Mongo driver knows how to interpret the `operation` objects (implemented in `processor/target/mongo.js`). <br>• Date fields are supplied in a format parseable by `new Date()`. |

---

## 4. Integration Points  

| Component | How it uses this schema |
|-----------|------------------------|
| `processor/main.js` | Orchestrates the end‑to‑end job; loads the schema to initialise the target driver. |
| `processor/source/oracle.js` | Emits rows whose column names must match the `fields.name` values. |
| `processor/target/mongo.js` | Consumes `schema` to (a) drop/create the collection, (b) build indexes, (c) map each source row to a Mongo document applying the defined operations. |
| `stack/msps/msps2ExecutionQueue.js` | Triggers the job (e.g., via a message queue). The queue payload may reference the schema name (`sims`). |
| `stack/vaz_sync.yaml` (Docker‑Compose) | Deploys the container that runs the Node process; environment variables for Mongo are injected here. |
| `config/production.json` | Holds DB connection details, batch sizes, and possibly the `lossPerc` override. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Full collection drop** (`dropCollection: true`) | Total loss of previously loaded data if the job crashes after the drop. | • Run the job in a *staging* environment first.<br>• Add a pre‑run backup step (e.g., `mongodump`).<br>• Consider switching to `dropCollection: false` and using upserts after validation. |
| **Loss tolerance breach** (`lossPerc: 0.1`) | Job aborts if >10 % of rows are rejected, potentially leaving partial data. | • Monitor the rejection count in logs.<br>• Tune `lossPerc` per SLA.<br>• Implement detailed validation logs to quickly identify problematic rows. |
| **Ignored fields** (`operation: ignore`) | Important data may be silently omitted if schema diverges from source. | • Periodic schema review against source table definitions.<br>• Add unit tests that assert required columns are persisted. |
| **Array merge collisions** (`mergeArray`) | Duplicate values may appear in `msisdns` array. | • Deduplicate in the `mergeArray` implementation or add a post‑process step. |
| **Index creation failure** | If the `sponsor` index cannot be built (e.g., due to existing data), the job may fail. | • Ensure the index definition matches the final document shape.<br>• Log index creation status and retry on transient errors. |

---

## 6. Running / Debugging the Schema  

1. **Standard execution** (triggered by the queue):  
   ```bash
   export NODE_ENV=production
   export MONGO_URI=mongodb://user:pwd@mongo-host:27017
   export MONGO_DB=msps_prod
   node processor/main.js   # reads the schema automatically
   ```

2. **Manual test of the schema mapping** (useful for developers):  
   ```js
   const { schema } = require('./stack/msps/msps2MongoSchema');
   const mongoDriver = require('../processor/target/mongo'); // assumed path
   // Mock a source row
   const row = {
     iccid: 123456789012345,
     primaryImsi: '001010123456789',
     msisdn1: '447700900123',
     msisdn2: null,
     // … other columns …
   };
   const doc = mongoDriver.mapRowToDoc(row, schema.sims);
   console.log(JSON.stringify(doc, null, 2));
   ```

3. **Debugging tips**  
   * Set `DEBUG=processor:*` (or the logger used in the project) to see per‑field operation logs.  
   * Verify that the generated document contains the expected `sponsors`, `bizUnits`, `msisdns`, and `cust` structures.  
   * Use `mongosh` to inspect the collection after a run: `db.sims.findOne({iccid: <value>})`.

---

## 7. External Configuration / Environment Variables  

| Source | Usage |
|--------|-------|
| `move-MSPS/config/production.json` | Supplies Mongo connection string (`mongo.uri`), database name, and possibly overrides for `dropCollection` or `lossPerc`. |
| Environment variables (`MONGO_URI`, `MONGO_DB`) | May be read by the generic Mongo driver if the config file references them (common pattern in this codebase). |
| Queue/message payload (from `msps2ExecutionQueue.js`) | May contain runtime flags (e.g., `dryRun`) that affect whether the schema’s `dropCollection` is honoured. |

---

## 8. Suggested Improvements (TODO)

1. **Add schema validation layer** – before inserting a document, run it through a JSON‑Schema validator derived from this definition to catch type mismatches early.
2. **Make `dropCollection` configurable at runtime** – expose a flag (e.g., `--recreate`) so that production runs can opt for incremental upserts instead of full drops, reducing data‑loss risk.  

---