**File:** `move-Inventory\move-MSPS\stack\msps\mspsMongoSchema.js`  

---

## 1. High‑Level Summary
This module defines the **MongoDB schema catalogue** used by the MSPS data‑move pipeline. It describes collections (`organizations`, `networks`, `pointCodes`, `tclPointCodes`, `imsiGt`, `msisdnGt`) together with their primary keys, field types, transformation operations (replace, lookup, exists), replacement rules (regex → function), and collection‑level options (indexes, drop‑collection flags, loss‑percentage tolerances). The schema is exported for consumption by the generic Mongo target processor (`move-Inventory\move-MSPS\processor\target\mongo.js`) and by execution‑queue scripts that orchestrate the load‑transform‑store workflow.

---

## 2. Important Functions & Data Structures  

| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `toUpper(x)` | function | Returns `x` in **UPPERCASE**. Used by replacement rules for abbreviations and parenthesised tokens. |
| `toProper(x)` | function | Capitalises the first character, lower‑cases the rest (Title‑Case). Applied to multi‑word tokens that are not abbreviations. |
| `toLower(x)` | function | Returns `x` in **lowercase**. Used for prepositions in replacement rules. |
| `schema` | plain object | Central definition of all Mongo collections handled by the MSPS pipeline. Each top‑level key (e.g. `organizations`) contains: <br>• `_id` – array of field names that compose the Mongo primary key.<br>• `options` – collection‑wide settings (indexes, `dropCollection`, `lossPerc`).<br>• `disableRecords` – records that must be filtered out.<br>• `fields` – field‑level metadata (name, type, optional `operation`).<br>• `replacements` – regex‑based text normalisation applied before persisting. |
| `exports.schema` | export | Makes the `schema` object available to other Node modules (processor, queue, orchestration scripts). |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | • Raw rows streamed from the **source processor** (`oracle.js`) – typically Oracle result sets.<br>• Runtime configuration (Mongo connection string, DB name) supplied by the target processor (`mongo.js`). |
| **Outputs** | • MongoDB collections created/updated according to the schema (e.g. `organizations`, `networks`, `pointCodes`, …).<br>• Indexes defined in `options.indexes` are built automatically by the target processor.<br>• Records filtered out by `disableRecords` are never persisted. |
| **Side‑effects** | • When `options.dropCollection` is true, the target processor will **drop** the existing collection before loading (potential data loss).<br>• `lossPerc` values are used by the processor to log/alert if the percentage of discarded records exceeds the tolerance. |
| **External Services / Collections** | • **Lookup collections** referenced in `operation.type: 'lookup'` : `countriesFinal`, `organizations`, `tclPointCodes`.<br>• **Existence check** (`operation.type: 'exists'`) against `tclPointCodes`.<br>• MongoDB server (host/port) – configured outside this file (environment variables or Docker‑compose secrets). |
| **Assumptions** | • All lookup collections are already materialised in the same Mongo database before this schema is applied.<br>• Field names used in lookups (`tcl`, `orgNo`, `_id`) exist and are indexed in the remote collections.<br>• Regex patterns are valid JavaScript RegExp strings; they are applied globally by the processor. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Component | Connection Detail |
|-----------|-------------------|
| **Source Processor (`oracle.js`)** | Reads Oracle tables, maps column names to the field names defined in `schema.<collection>.fields`. |
| **Target Processor (`mongo.js`)** | Imports `schema` via `require('.../mspsMongoSchema.js')`. Uses it to: <br>• Create/drop collections, <br>• Build indexes, <br>• Apply per‑field `operation` (replace, lookup, exists), <br>• Run the `replacements` pipeline on string fields before insertion. |
| **Execution Queues (`mspsExecutionQueue.js`, `msps2ExecutionQueue.js`)** | Queue workers reference the same schema to validate payloads and to decide which collection a given job should write to. |
| **Docker‑Compose Stack (`vaz_sync.yaml`)** | Provides the Mongo service, environment variables (e.g. `MONGO_URI`) that the target processor consumes. The schema file is mounted into the container image as part of the application code. |
| **Other Schema Files (`maximityOrganizationSchema.js`, `msps2MongoSchema.js`)** | Follow the same structural pattern; they are interchangeable for different data domains. Consistency across schemas is assumed by shared utility code in the processor layer. |
| **Lookup Data Loaders** | Separate scripts (not shown) populate `countriesFinal`, `organizations`, `tclPointCodes`. Those scripts must run **before** any pipeline that uses this schema. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental collection drop** (`dropCollection: true`) | Full loss of existing data if the pipeline is re‑run unintentionally. | • Gate the pipeline behind a manual “approve” step in CI/CD.<br>• Keep a recent backup of the Mongo database (e.g., `mongodump`). |
| **Loss‑percentage tolerance breach** (`lossPerc`) | Silent data quality degradation if many records are filtered out. | • Implement alerting on the processor’s loss‑percentage metric.<br>• Log the exact records that were discarded for post‑mortem. |
| **Invalid regex / replacement function** | Runtime errors or malformed data (e.g., `toProper` called on empty string). | • Add unit tests for each replacement rule.<br>• Validate regex syntax at startup (try‑catch around `new RegExp`). |
| **Missing lookup collections** | `lookup` operations return `null`, causing downstream failures or incorrect ISO codes. | • Enforce a pre‑flight check that all remote collections exist and contain required indexes.<br>• Fail fast with a clear error message if a lookup target is absent. |
| **Index creation failure** | Slow queries, possible pipeline timeouts. | • Ensure the Mongo user has `createIndex` privileges.<br>• Log index creation results; retry on transient errors. |
| **Schema drift** (field type mismatch between source and schema) | Data insertion errors, type coercion issues. | • Add schema validation step in the processor (e.g., using `ajv` or MongoDB schema validation). |

---

## 6. Running / Debugging the Schema

1. **Typical execution** (via the pipeline):  
   ```bash
   # In the Docker container or local dev environment
   npm run start:msps   # triggers the orchestrator which loads oracle.js → transforms → mongo.js
   ```
   The orchestrator loads this file automatically; no direct command is required.

2. **Standalone sanity check** (debugging):  
   ```javascript
   // testSchema.js
   const { schema } = require('./stack/msps/mspsMongoSchema');
   console.log(JSON.stringify(schema, null, 2));
   // optional: invoke a sample replacement
   const { toProper } = require('./stack/msps/mspsMongoSchema');
   console.log(toProper('example')); // => Example
   ```
   Run with `node testSchema.js` to verify the exported object and utility functions.

3. **Unit‑test example** (using Jest or Mocha):  
   ```javascript
   const { schema } = require('../mspsMongoSchema');
   test('organizations replacement – proper case', () => {
     const rule = schema.organizations.replacements[0];
     const re = new RegExp(rule.get, 'g');
     const input = 'ALPHA beta';
     const output = input.replace(re, rule.set);
     expect(output).toBe('Alpha Beta');
   });
   ```

4. **Logging**: The target processor (`mongo.js`) typically logs each transformation step. Increase log level (`DEBUG`) to see per‑field `operation` and replacement results.

---

## 7. External Configuration / Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| `MONGO_URI` (or similar) | Connection string for the Mongo instance – supplied by Docker‑compose or the runtime environment. |
| `DB_NAME` | Target database name – also external to this module. |
| **No direct config** is referenced inside `mspsMongoSchema.js`; all runtime parameters are injected by the consumer (processor or queue). |

If the organization wishes to make the schema configurable (e.g., toggle `dropCollection` per environment), they could externalise those flags into a JSON/YAML config file read at module init.

---

## 8. Suggested TODO / Improvements

1. **Add schema validation tests** – automate verification that every `operation.lookup.remote` collection exists and that the referenced fields are indexed.  
2. **Externalise replacement patterns** – move regex strings and their associated transformation functions into a separate JSON/YAML file so that business users can adjust text‑normalisation rules without code changes.  

---