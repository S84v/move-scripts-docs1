**File:** `move-Inventory\move-MSPS\stack\vaz_sync.yaml`  
**Type:** Docker‑Compose (v3.7) stack definition for production synchronizer services.

---

## 1. High‑Level Summary
This YAML file defines a Docker Swarm stack named **sync** that deploys three identical Node.js synchronizer containers (MSPS, Tableau, MaximityOrganization). Each container runs the same `vaz1:5000/sync:v1.0.0` image, which houses the “move‑Inventory → move‑MSPS” codebase (processor, source/oracle, target/mongo, etc.). The stack injects service‑specific configuration, execution‑queue definitions, and MongoDB schema files via Docker **configs**, mounts host log directories, and supplies runtime environment variables (e.g., `NODE_ENV=production`). The containers connect to Oracle source databases (via `extra_hosts`) and write transformed data into MongoDB collections, producing operational logs under `/logs/sync/<service>`.

---

## 2. Core Components & Responsibilities

| Component | Responsibility |
|-----------|-----------------|
| **services.msps** | Runs the MSPS data‑move job; reads `msps.json` config, queue, and schema; writes logs to `/logs/sync/msps`. |
| **services.tableau** | Runs the Tableau synchronizer; distinguished by `SERVICE=sync_tableau` env var; uses `tableau.json` config, queue, schema; logs to `/logs/sync/tableau`. |
| **services.maximityorganization** | Runs the Maximity Organization synchronizer; uses `maximityOrganization.json` config, queue, schema; logs to `/logs/sync/maximityorganization`. |
| **configs.\*.config** | JSON files containing service‑specific runtime settings (DB connection strings, collection names, batch sizes, etc.). |
| **configs.\*.queue** | JavaScript files (`executionQueue.js`) that define the job‑queue implementation (e.g., Bull, RabbitMQ) used by the Node app to schedule/track moves. |
| **configs.\*.schema** | MongoDB schema definitions (`mongoSchema.js`) that the target processor validates against before persisting data. |
| **environment.NODE_ENV** | Forces the Node process into production mode (`npm run production`). |
| **environment.SERVICE** (Tableau only) | Allows the same image to branch logic based on service name. |
| **volumes.logtmp** | Host‑mounted directory for rotating log files; also used by the Node app for temporary files. |
| **volumes.local_host_cfg** | Shared host configuration (e.g., SSH keys, host‑specific overrides) required by source/oracle module. |
| **extra_hosts** | Provides static DNS entries for Oracle hosts (`camttppdb003`) that the source module resolves. |
| **deploy.resources** | CPU & memory limits/reservations to protect the Swarm node from overload. |
| **deploy.placement.constraints** | Prevents deployment on node `mttvaz02` (likely a maintenance or non‑compatible host). |
| **command** | Executes `npm run production`, which starts the synchronizer’s main entry point (`processor/main.js`). |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Service‑specific JSON config files (`msps.json`, `tableau.json`, `maximityOrganization.json`).<br>• Execution‑queue JavaScript (`*ExecutionQueue.js`).<br>• MongoDB schema JavaScript (`*MongoSchema.js`).<br>• Oracle host IPs via `extra_hosts`.<br>• Environment variables (`NODE_ENV`, optional `SERVICE`). |
| **Outputs** | • Log files under `/logs/sync/<service>` (rotated by the container).<br>• Data written to MongoDB collections defined in the schema files.<br>• Optional audit records or metrics emitted by the Node app (depends on internal implementation). |
| **Side Effects** | • Opens TCP connections to Oracle source databases (read).<br>• Performs inserts/updates on MongoDB (write).<br>• May enqueue/dequeue jobs in a message broker if the queue implementation uses an external service (e.g., Redis, RabbitMQ). |
| **Assumptions** | • Docker Swarm is already initialized and the image `vaz1:5000/sync:v1.0.0` is available in the private registry.<br>• Host paths (`/logs/sync/*`, `/lz/local_host_cfg`) exist and are writable by the Docker daemon.<br>• The Oracle hosts (`camttppdb003`) are reachable from the Swarm nodes.<br>• MongoDB connection details are embedded in the JSON config files (or pulled from a secret not shown here). |

---

## 4. Integration Points with the Rest of the System

| Integration | How It Connects |
|-------------|-----------------|
| **move‑Inventory → move‑MSPS codebase** | The container image includes `processor/main.js`, `processor/source/oracle.js`, `processor/target/mongo.js`, and the `config` directory referenced by the injected configs. |
| **External Oracle DB** | Resolved via `extra_hosts`; the source module uses the connection strings from the injected JSON config to pull inventory data. |
| **MongoDB Cluster** | Target module (`processor/target/mongo.js`) reads the schema config to validate and write documents. |
| **Job Queue Service** | The injected `executionQueue.js` likely creates a client to a broker (Redis/RabbitMQ). The rest of the codebase expects this module to expose `enqueue`, `dequeue`, etc. |
| **Logging & Monitoring** | Host‑mounted `/logs/sync/*` directories are consumed by external log aggregation (e.g., Splunk, ELK). |
| **Configuration Management** | All service‑specific JSON files live under `/app/docker-configs/stack/sync/...` and are version‑controlled separately from the stack definition. |
| **Other Stacks** | The same image is reused for other synchronizer stacks (e.g., `vaz_sync.yaml` may be part of a larger orchestrated deployment that includes other services like `vaz_ingest`, `vaz_export`). |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale or mismatched config files** (e.g., JSON schema not aligned with code) | Automate config validation in CI; version‑lock configs with the image tag; run a pre‑deployment health‑check script. |
| **Network connectivity loss to Oracle hosts** | Implement retry/back‑off in `oracle.js`; monitor connectivity via a side‑car health‑check container. |
| **Resource contention** (CPU/Memory limits too low) | Review actual usage in production; adjust `limits`/`reservations` accordingly; enable Docker Swarm auto‑scaling if available. |
| **Log volume overflow** (host directory fills) | Rotate logs inside the container; set host‑level logrotate; monitor disk usage alerts. |
| **Deployment on prohibited node** (constraint mis‑configuration) | Verify node labels and constraints before stack update; use `docker node ls` to confirm node names. |
| **Credentials exposure** (DB passwords in JSON) | Move secrets to Docker Swarm secrets and reference them in the config files; avoid plain‑text passwords. |
| **Container start‑up failures** (missing volume, config) | Add `depends_on` logic in a wrapper script; validate volume paths and config existence before `docker stack deploy`. |

---

## 6. Running / Debugging the Stack

1. **Deploy** (as indicated in the comment):  
   ```bash
   docker stack deploy -c vaz_sync.yaml sync
   ```

2. **Check service status**:  
   ```bash
   docker stack services sync
   ```

3. **View logs (real‑time)**:  
   ```bash
   docker service logs -f sync_msps
   docker service logs -f sync_tableau
   docker service logs -f sync_maximityorganization
   ```

4. **Inspect a running container** (e.g., to view environment or file contents):  
   ```bash
   CONTAINER=$(docker ps --filter "name=sync_msps" -q | head -n1)
   docker exec -it $CONTAINER /bin/sh
   ```

5. **Force a rolling update** (e.g., after changing a config file):  
   ```bash
   docker service update --force sync_msps
   ```

6. **Debugging tip** – If the Node process exits early, check the exit code and the `npm run production` script in `package.json`. Adding `--inspect` to the command (`node --inspect index.js`) can attach a debugger.

---

## 7. External Config / Environment References

| Reference | Path | Purpose |
|-----------|------|---------|
| **JSON configs** | `/app/docker-configs/stack/sync/msps.json` (and tableau/maximity equivalents) | Holds runtime parameters: Oracle connection strings, MongoDB URI, batch sizes, etc. |
| **Execution queue scripts** | `/app/docker-configs/stack/sync/mspsExecutionQueue.js` (and equivalents) | Provides the queue client implementation used by the synchronizer. |
| **Mongo schema scripts** | `/app/docker-configs/stack/sync/mspsMongoSchema.js` (and equivalents) | Defines collection validation rules and field mappings. |
| **Host‑mounted logs** | `/logs/sync/<service>` | Persistent storage for log files; consumed by external log collectors. |
| **Local host config** | `/lz/local_host_cfg` | May contain SSH keys, Oracle client libraries, or other host‑specific overrides required by `oracle.js`. |
| **Environment variables** | `NODE_ENV=production`, optional `SERVICE=sync_tableau` | Controls Node.js mode and service‑specific branching logic. |

---

## 8. Suggested Improvements (TODO)

1. **Introduce Docker Secrets for DB credentials** – Move any passwords or private keys out of the JSON configs into Swarm secrets and reference them at runtime to reduce the risk of credential leakage.

2. **Add a health‑check** to each service (e.g., `curl http://localhost:3000/health`) so Swarm can automatically restart unhealthy containers and provide visibility in `docker service ps`. 

---