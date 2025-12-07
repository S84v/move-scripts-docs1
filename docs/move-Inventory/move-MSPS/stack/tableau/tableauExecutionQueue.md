**File:** `move-Inventory\move-MSPS\stack\tableau\tableauExecutionQueue.js`  

---

## 1. High‑level Summary
This module defines the Oracle‑source queue configuration for the Tableau integration layer. It exports a single constant, `oracleQueue`, which is an array of objects describing the SQL query to extract the *Tableau User Mapping* view from the Oracle database. The exported definition is consumed by the generic “execution‑queue” runner (e.g., `move‑Inventory\move‑MSPS\stack\tableau\tableauExecutionQueueRunner.js` or a shared queue processor) to pull user‑mapping rows, transform them, and forward them to downstream services such as MongoDB collections, APIs, or SFTP feeds.

---

## 2. Key Elements

| Symbol / Export | Type | Responsibility |
|-----------------|------|----------------|
| `oracleQueueTableau` | `Array<Object>` | Holds one or more queue descriptors. Each descriptor contains:<br>• `name` – human‑readable identifier.<br>• `target` – logical destination used by downstream processors (e.g., Mongo collection name).<br>• `query` – the exact Oracle SQL to execute. |
| `exports.oracleQueue` | Export | Makes the queue definition available to the runtime orchestrator. |

*No functions are defined in this file; it is a pure configuration module.*

---

## 3. Inputs, Outputs, Side‑effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • Oracle database connection (host, port, service name, credentials) – supplied by environment variables or a shared `dbConfig.js` module.<br>• The `Usermappingview` view must exist in schema `Tableaureporting` with columns `Userid`, `Secsid`, `Buid`, `Usertype`. |
| **Outputs** | The execution engine receives a result set of rows with fields:<br>`userId`, `secsId`, `buId`, `userType`.<br>These rows are then routed to the logical target `tableauUserMapping` (typically a MongoDB collection defined in `tableauMongoSchema.js`). |
| **Side‑effects** | None directly; the module only provides metadata. Side‑effects occur in the consumer (e.g., inserting into Mongo, publishing to a queue). |
| **Assumptions** | • Oracle client libraries are correctly installed in the container running the queue processor.<br>• Network connectivity to the Oracle host is allowed from the container.<br>• The downstream schema (`tableauUserMapping`) matches the field names produced by the query. |

---

## 4. Integration Points

| Component | Relationship |
|-----------|--------------|
| **`tableauExecutionQueueRunner.js` (or similar)** | Imports `oracleQueue` to build the execution plan. |
| **`move‑Inventory\move‑MSPS\stack\tableau\tableau.json`** | Holds service‑level configuration (e.g., Docker‑Compose service name, environment variables). |
| **`move‑Inventory\move‑MSPS\stack\tableau\tableauMongoSchema.js`** | Defines the MongoDB collection (`tableauUserMapping`) that receives the transformed rows. |
| **Shared queue framework** (e.g., `move‑Inventory\move‑MSPS\stack\msps\mspsExecutionQueue.js`) | Uses the same export pattern (`exports.oracleQueue`) – this file follows that convention, enabling the generic runner to treat Tableau queues identically to MSPS or Maximity queues. |
| **Docker‑Compose stack (`vaz_sync.yaml`)** | Deploys the Tableau queue processor container, injecting DB credentials via environment variables (`ORACLE_USER`, `ORACLE_PASS`, etc.). |
| **External services** | Oracle database (source), MongoDB (target), optional message broker (e.g., RabbitMQ) if the runner publishes events. |

---

## 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Schema drift** – columns renamed or view removed. | Add a health‑check script that validates the query returns the expected columns before processing. |
| **Large result set** – full table scan could exhaust memory or cause long runtimes. | Implement pagination (`ROWNUM`/`OFFSET FETCH`) or add a `WHERE` clause with a timestamp/ID filter. |
| **Credential leakage** – DB passwords stored in plain text. | Use Docker secrets or a vault (e.g., HashiCorp Vault) and reference via environment variables only. |
| **Uncaught query errors** – syntax error or connectivity loss stops the whole pipeline. | Ensure the generic runner wraps query execution in try/catch and retries with exponential back‑off; surface errors to monitoring (Prometheus/ELK). |
| **Missing target collection** – `tableauUserMapping` not created. | Include a startup migration step that creates the collection and indexes if absent. |

---

## 6. Running / Debugging the Queue

1. **Start the stack** (assuming Docker‑Compose):  
   ```bash
   cd move-Inventory/move-MSPS/stack
   docker compose -f vaz_sync.yaml up -d tableau-queue
   ```

2. **Manual execution (for debugging):**  
   ```bash
   # Inside the container or local dev environment
   node -e "const {oracleQueue}=require('./tableau/tableauExecutionQueue'); console.log(JSON.stringify(oracleQueue, null, 2));"
   ```

3. **Validate the query against Oracle:**  
   ```bash
   sqlplus $ORACLE_USER/$ORACLE_PASS@$ORACLE_TNS <<SQL
   ${oracleQueue[0].query};
   SQL
   ```

4. **Check downstream data:**  
   ```bash
   mongo --eval "db.tableauUserMapping.find().limit(5).pretty()"
   ```

5. **Log inspection:**  
   The generic runner writes to stdout and to a file mounted at `/var/log/tableau-queue.log`. Tail it to see success/failure counts.

---

## 7. External Configuration & Environment Variables

| Variable | Purpose |
|----------|---------|
| `ORACLE_HOST`, `ORACLE_PORT`, `ORACLE_SERVICE`, `ORACLE_USER`, `ORACLE_PASS` | Connection details for the Oracle source. |
| `MONGO_URI` | MongoDB connection string used by the downstream writer. |
| `TABLEAU_QUEUE_BATCH_SIZE` (optional) | Controls how many rows the runner fetches per batch. |
| `TABLEAU_QUEUE_CRON` (optional) | Cron expression for scheduled execution (if the runner is time‑driven). |

These variables are typically defined in the Docker‑Compose file (`vaz_sync.yaml`) under the Tableau service’s `environment:` block.

---

## 8. Suggested Improvements (TODO)

1. **Add pagination support** – Refactor the `query` to include a bind variable for the last processed primary key and expose `batchSize` via config, preventing unbounded memory usage.
2. **Externalize the query** – Move the SQL string to a separate `.sql` file or a configuration JSON so that DBAs can modify it without redeploying code; load it at runtime with `fs.readFileSync`. 

---