**File:** `move-Inventory\move-MSPS\config\mongoSchema.js`  

---

## 1. High‑Level Summary
This module defines the **canonical MongoDB schema** used by the *move‑MSPS* data‑move pipeline. For each logical entity (organizations, networks, point codes, GT tables, etc.) it declares the collection’s primary key, field definitions, transformation rules, lookup relationships, and collection‑level options (indexes, drop‑and‑re‑create flags, loss‑tolerance percentages). The exported `schema` object is consumed by the generic ETL engine (see `app.js` and the execution‑queue configuration) to drive validation, field‑level replace/lookup operations, and collection creation before loading transformed data into the target MongoDB instance.

---

## 2. Important Constructs

| Construct | Type | Responsibility |
|-----------|------|----------------|
| `toUpper(x)` | function | Upper‑cases the supplied string (used by replacement rules). |
| `toProper(x)` | function | Capitalises the first character, lower‑cases the rest (proper‑case). |
| `toLower(x)` | function | Lower‑cases the supplied string. |
| `schema` | plain object (exported) | Holds per‑collection configuration: primary key (`_id`), field list, transformation operations, replacement regexes, collection options (indexes, drop flags, loss tolerance). |
| `module.exports.schema` | export | Makes the schema available to the runtime engine (`require('./mongoSchema')`). |

*Each collection entry (e.g., `organizations`, `networks`, `pointCodes`, `imsiGt`, `msisdnGt`) follows the same shape:*

- **`_id`** – array of field names that compose the MongoDB document `_id`.
- **`options`** – optional collection‑level settings (`dropCollection`, `lossPerc`, `indexes`).
- **`disableRecords`** – criteria for records that must be filtered out.
- **`fields`** – array of field descriptors (`name`, `type`, optional `operation`).
- **`replacements`** – regex‑based string transformations applied before persisting.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | The schema is *static* – no runtime input. It is read by the ETL engine together with the source data files (CSV, flat files, etc.) that are being moved. |
| **Outputs** | The engine uses the schema to generate MongoDB collection definitions, apply field‑level transformations, and finally insert the transformed documents into the target MongoDB cluster. |
| **Side‑Effects** | When `options.dropCollection: true` is set, the engine will **drop and recreate** the collection, potentially deleting existing data. |
| **Assumptions** | - A higher‑level driver (e.g., `app.js`) interprets the schema format. <br>- Lookup targets (`countriesFinal`, `organizations`, `tclPointCodes`) exist in the same MongoDB database and are populated before dependent collections are processed. <br>- Regex strings are valid JavaScript RegExp patterns. <br>- The environment provides a MongoDB connection string via configuration (`config/development.json` or similar). |

---

## 4. Connection to Other Scripts & Components

| Component | How it uses `mongoSchema.js` |
|-----------|------------------------------|
| **`move-MSPS/app.js`** | Imports the schema (`const { schema } = require('./config/mongoSchema');`) and passes it to the generic `move` engine that orchestrates extraction, transformation, and loading. |
| **`config/config.js` / `config/development.json`** | Supplies DB connection details and environment flags that the engine uses together with the schema to open the target MongoDB instance. |
| **`config/executionQueue.js`** | Defines the order of processing; each step references a collection name that must match a key in `schema`. |
| **Utility modules (`utils/pgSql.js`, `utils/utils.js`)** | May provide source data (PostgreSQL extracts) that are later transformed according to the rules defined here. |
| **Test harness (`test.js`)** | Loads the schema to validate that transformation rules behave as expected. |
| **External services** | Lookup operations rely on other collections (`countriesFinal`, `organizations`, `tclPointCodes`) that are populated by separate ingestion jobs (e.g., country master data loader). |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Drop‑Collection flag** (`options.dropCollection: true`) | Permanent loss of existing data if the job is re‑run unintentionally. | Guard the job with a *dry‑run* mode; require explicit `--force-drop` CLI flag; snapshot the collection before dropping. |
| **Loss‑percentage tolerance** (`lossPerc`, `lossTolerance`) | Silent acceptance of data loss beyond the configured threshold. | Emit warnings when actual loss exceeds the threshold; abort the job if loss > configured value. |
| **Regex / replacement errors** | Incorrect string mutation (e.g., over‑capitalisation) leading to downstream validation failures. | Unit‑test each replacement rule; log before/after values for a sample of records. |
| **Lookup failures** (missing foreign key) | `null` values inserted for `iso` or `isoOrg`, breaking downstream reporting. | Validate lookup existence; fallback to a default code; raise an alert if lookup count > 0. |
| **Schema drift** (field type changes) | Runtime type errors when the engine attempts to cast values. | Version the schema file; enforce CI linting that checks for breaking changes. |

---

## 6. Running / Debugging the File

1. **Typical execution** (via the pipeline):  
   ```bash
   NODE_ENV=development node app.js   # app.js loads mongoSchema.js internally
   ```
2. **Standalone sanity check** (quickly inspect the exported object):  
   ```bash
   node -e "console.log(require('./config/mongoSchema').schema)"
   ```
3. **Debugging transformations**:  
   - Insert a temporary `console.log` inside a replacement rule’s `set` function (e.g., `toProper`) to view the input/output.  
   - Use the test harness (`npm test` or `node test.js`) which loads the schema and runs predefined assertions.  
   - Enable verbose logging in the ETL engine (often via an env var like `LOG_LEVEL=debug`).  

---

## 7. External Configuration / Environment Variables

| Reference | Purpose |
|-----------|---------|
| `config/development.json` (or other env‑specific JSON) | Holds MongoDB URI, authentication, and possibly a flag that toggles `dropCollection` behaviour. |
| `process.env.NODE_ENV` | Determines which config file is loaded; the schema itself is environment‑agnostic but the consuming engine may apply different options per environment. |
| No direct env vars are read inside `mongoSchema.js`; all external parameters are injected by the caller. |

---

## 8. Suggested Improvements (TODO)

1. **Schema Validation Layer** – Introduce a JSON‑Schema definition for each collection and run it against the `schema` object at startup to catch malformed entries (missing `type`, invalid regex, etc.).
2. **Externalise Replacement Rules** – Move the regex patterns and associated transformation functions to a separate JSON/YAML file so that business users can adjust wording rules without code changes, and load them dynamically.  

---