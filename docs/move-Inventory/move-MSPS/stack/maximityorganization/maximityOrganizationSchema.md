**File:** `move-Inventory\move-MSPS\stack\maximityorganization\maximityOrganizationSchema.js`

---

## 1. High‑Level Summary
This module defines the **MongoDB schema** for the `organizations` collection used by the “maximity organization” data‑move stack. It specifies primary key(s), collection options (indexes, drop flag), record‑level disabling rules, field definitions, and a set of **text‑normalisation replacement rules** that are applied to the `orgName` field during the transformation phase. The schema is consumed by the generic Mongo target processor (`processor/target/mongo.js`) to create the collection, enforce indexes, perform look‑ups (e.g., country ISO mapping), and apply the defined string‑case transformations.

---

## 2. Important Functions & Objects  

| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `toUpper(x)` | function | Returns `x` in **UPPERCASE**. |
| `toProper(x)` | function | Capitalises the first character, lower‑cases the rest (Title‑Case). |
| `toLower(x)` | function | Returns `x` in **lowercase**. |
| `schema` | constant object | Full declarative description of the `organizations` collection: primary key, indexes, disabled records, field metadata, and replacement rules. |
| `exports.schema` | export | Makes the `schema` object available to other modules (e.g., the Mongo target processor). |

---

## 3. Responsibilities of the Schema Elements  

| Section | Purpose |
|---------|---------|
| `_id: ['orgNo']` | Declares `orgNo` as the document primary key. |
| `options.indexes` | Creates a single ascending index on the `tcl` field (`name: 'TCL'`). |
| `disableRecords` | Filters out any source rows where `orgName` equals `'UNKNOWN'`. |
| `fields` | Describes each column: data type, and any special operation (`replace`, `lookup`). |
| `operation.type: 'lookup'` (field `iso`) | Performs a foreign‑key style lookup into the `countriesFinal` collection, matching on `tcl` and returning the `_id` of the country. |
| `replacements` | Ordered regex‑based transformation rules applied to `orgName` before persisting: proper‑casing, lower‑casing of prepositions, upper‑casing of known corporate suffixes, and upper‑casing of short words inside parentheses. |

---

## 4. Inputs, Outputs, Side Effects & Assumptions  

| Aspect | Detail |
|--------|--------|
| **Inputs** | - Source rows supplied by the Oracle source processor (`processor/source/oracle.js`). <br> - Environment/connection details for Mongo (handled elsewhere). |
| **Outputs** | - A MongoDB collection named `organizations` populated with transformed documents. <br> - Index `TCL` on `tcl`. |
| **Side Effects** | - May drop the collection if `dropCollection: true` is uncommented. <br> - Inserts/updates documents in the target Mongo instance. |
| **Assumptions** | - The remote collection `countriesFinal` exists and contains a `tcl` field and `_id` to resolve the `iso` lookup. <br> - Source data contains the fields `orgNo`, `orgName`, `tcl`. <br> - Regex patterns are compatible with JavaScript `RegExp` engine. |

---

## 5. Integration Points  

| Component | How it Connects |
|-----------|-----------------|
| `processor/target/mongo.js` | Imports `schema` via `require('.../maximityOrganizationSchema.js')` and drives collection creation, index building, and per‑field operations (replace, lookup, replacements). |
| `processor/main.js` | Orchestrates the overall move; selects the appropriate schema based on the stack being executed (`maximityorganization`). |
| `stack/vaz_sync.yaml` (Docker‑Compose) | Deploys the container that runs the move; mounts this file into the container so the target processor can read it at runtime. |
| `stack/maximityorganization/maximityOrganizationExecutionQueue.js` | May reference the same collection name (`organizations`) when queuing downstream jobs; relies on the schema’s primary key (`orgNo`) for idempotency. |
| `config/*.json` (development/production) | Supplies Mongo connection strings, Oracle credentials, and feature flags that affect whether the schema is applied (e.g., a flag to enable `dropCollection`). |

---

## 6. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect regex replacement** – over‑matching could corrupt organization names. | Data quality issue, downstream reporting errors. | Add unit tests for each regex against a representative sample set; log original vs. transformed values at DEBUG level. |
| **Missing `countriesFinal` collection** – lookup for `iso` fails, causing null or error. | Incomplete records, possible job abort. | Validate existence of the lookup collection during job start; fallback to a default ISO code if not found. |
| **Index conflict** – if another service creates an index named `TCL` with different spec. | Index build failure, job stop. | Prefix index names with stack identifier (e.g., `maxOrg_TCL`) or enforce unique index naming via a CI check. |
| **Accidental collection drop** – uncommented `dropCollection: true` in production. | Total data loss. | Guard the flag with an environment variable (`DROP_COLLECTION`) that defaults to `false` in production; add a pre‑run confirmation prompt in CI. |
| **Schema drift** – field type changes not reflected in downstream consumers. | Runtime type errors. | Version the schema file (e.g., `v1.2.maximityOrganizationSchema.js`) and enforce compatibility checks in the orchestrator. |

---

## 7. Running / Debugging the Schema  

1. **Standard execution** (via the orchestrator):  
   ```bash
   npm run move -- --stack maximityorganization --env production
   ```
   The orchestrator loads `maximityOrganizationSchema.js` automatically.

2. **Isolated validation** (quick sanity check):  
   ```js
   // testSchema.js
   const { schema } = require('./stack/maximityorganization/maximityOrganizationSchema');
   console.log(JSON.stringify(schema, null, 2));
   ```
   Run with `node testSchema.js` to verify the exported object.

3. **Debugging replacements**:  
   ```js
   const { schema } = require('./.../maximityOrganizationSchema');
   const sample = { orgName: 'Plc de la (abc) Gmbh' };
   const repl = schema.organizations.replacements;
   let name = sample.orgName;
   repl.forEach(r => {
       const regex = new RegExp(r.get, 'g');
       name = name.replace(regex, r.set);
   });
   console.log(name);
   ```
   Compare the output with expected transformation.

4. **Log inspection**: Enable verbose logging in `processor/target/mongo.js` (e.g., set `LOG_LEVEL=debug`) to see each replacement applied.

---

## 8. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| `process.env.MONGO_URI` (or similar) | Connection string for the target Mongo instance – used by the generic Mongo processor, not directly in this file. |
| `process.env.DROP_COLLECTION` (optional) | If the schema’s `options.dropCollection` flag is toggled at runtime, this env var can control it safely. |
| `config/development.json` & `config/production.json` | Provide the stack name, execution queue settings, and feature toggles that determine whether this schema is loaded. |
| `stack/vaz_sync.yaml` | Docker‑Compose mounts the `maximityOrganizationSchema.js` file into the container; any volume path changes must be reflected here. |

---

## 9. Suggested Improvements (TODO)

1. **Unit‑test suite for replacement logic** – create a Jest/Mocha test file that exercises each regex against edge‑case strings and asserts the final `orgName`.  
2. **Externalise regex patterns** – move the `get` strings into a JSON configuration (`maximityOrganizationReplacements.json`) so they can be tuned without code changes and version‑controlled separately.  

---