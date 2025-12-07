**File:** `move-Inventory\move-KYC-and-Esimshub\deployment.yml`  

---

## 1. High‑Level Summary
This Kubernetes **Deployment** definition provisions a single‑replica pod that runs the *eKYC PostgreSQL input processor* (`app.js`) for a specific database (`{{dbname}}`) and table (`{{table}}`). The pod pulls a Docker image from a configurable container registry, mounts an Azure‑backed PersistentVolumeClaim (`vazindia`) at `/vaz`, and expects configuration files under `/app/config`. The container is started with `node app.js` in production mode and exposes port 8081 for internal service communication or health‑checking.

---

## 2. Key Kubernetes Objects & Their Responsibilities  

| Object | Responsibility |
|--------|-----------------|
| **Deployment** `pgsql-input-processor-ekyc-{{dbname}}-{{table}}` | Guarantees a single running replica, handles rolling updates, and ties together pod template, selector, and labels. |
| **Pod Template** (under `spec.template`) | Describes the container runtime environment for the processor. |
| **Container** `pgsql-input-processor-ekyc-{{dbname}}-{{table}}` | Executes the Node.js application (`app.js`) that reads from PostgreSQL, transforms KYC data, and forwards it to downstream systems (e.g., Azure storage, messaging queues). |
| **Env vars** `NODE_ENV`, `NODE_CONFIG_DIR` | Switches the app to production mode and points the `node-config` library to the runtime configuration directory. |
| **Resources** (requests/limits) | Guarantees 1.5 CPU / 3 GiB memory minimum, caps at 2 CPU / 4 GiB to protect the node. |
| **Image** `{{containerRegistry}}/{{repository}}-{{dbname}}-{{table}}:{{tag}}` | Pulls the version‑specific processor image; `imagePullPolicy: Always` forces a fresh pull on each pod start. |
| **Command** `["node","app.js"]` | Entry point for the Node.js process. |
| **Port** `8081` | Exposes the internal HTTP endpoint used by health checks or downstream services. |
| **VolumeMount** `/vaz` | Provides the container with access to Azure‑backed storage (PVC `vazindia`). |
| **PersistentVolumeClaim** `vazindia` | Binds the pod to a pre‑provisioned Azure Disk/Files share that stores large payloads or intermediate files. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables (`NODE_ENV`, `NODE_CONFIG_DIR`). <br>• Config files located in `/app/config` (e.g., DB connection strings, API keys). <br>• Docker image tag (`{{tag}}`). <br>• PVC `vazindia` providing Azure storage. |
| **Outputs** | • Processed KYC records sent to downstream systems (e.g., Azure Blob, Kafka, internal APIs). <br>• Logs written to stdout (captured by Kubernetes). |
| **Side‑Effects** | • Reads from a PostgreSQL source table (`{{dbname}}`.`{{table}}`). <br>• May write temporary files to `/vaz`. <br>• May invoke external services (REST APIs, message queues). |
| **Assumptions** | • The PVC `vazindia` exists and is bound to a suitable Azure storage class. <br>• The container registry is reachable from the cluster (network, auth). <br>• Required secrets (DB credentials, API tokens) are injected via the `node-config` files or Kubernetes Secrets (not shown here). <br>• The `app.js` code follows the conventions used in other `move-…` processors (e.g., uses `utils`, `mongo.js`, `mssql.js`). |

---

## 4. Integration Points with the Rest of the System  

| Component | Connection Mechanism |
|-----------|----------------------|
| **`move-KYC-and-Esimshub/app.js`** | This Deployment runs the same `app.js` entry point; the source code lives in the `move-KYC-and-Esimshub` folder and shares utility modules (`utils/*.js`). |
| **PostgreSQL source** | Configured via `node-config` (likely `config/default.json` or environment‑specific overrides). The processor performs delta or full sync similar to the `move-BOSS-Maximity` processors. |
| **Azure storage (PVC `vazindia`)** | Used for staging files that may later be uploaded to Azure Blob or consumed by other pipelines. |
| **Downstream services** | Not defined in this YAML but typical targets are: <br>• Azure Event Hub / Service Bus (for async processing). <br>• Internal REST endpoints (e.g., customer‑profile service). |
| **CI/CD pipeline** (`azure-pipelines.yml`) | The pipeline builds the Docker image, pushes it to `{{containerRegistry}}`, and then applies this Deployment (likely via Helm or `kubectl apply`). |
| **Other `move-…` processors** | May run in parallel for different tables; they share the same namespace and may coordinate via a shared database lock or message queue. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Image pull failures** (wrong registry credentials, tag not found) | Pod stays in `ImagePullBackOff`, no processing. | Use Kubernetes `imagePullSecrets`; validate tag existence in CI before deployment. |
| **PVC unavailable or full** | Processor cannot write temporary files → job stalls. | Monitor PVC capacity; set alerts; configure a `ReadWriteMany` Azure Files share with sufficient quota. |
| **Resource exhaustion** (CPU/Memory spikes) | OOM kill or throttling, data loss. | Add HorizontalPodAutoscaler (HPA) if workload is bursty; tune requests/limits based on profiling. |
| **Missing/incorrect config files** | Application crashes on start. | Include a ConfigMap/Secret volume mount for `/app/config`; add an init‑container that validates presence of required keys. |
| **Unobserved pod health** | Failures go unnoticed. | Add `livenessProbe` (e.g., HTTP GET `/healthz`) and `readinessProbe` to the container spec. |
| **Secret leakage via env vars** | Security breach. | Store credentials in Kubernetes Secrets; reference them via `envFrom` rather than plain env values. |

---

## 6. Running / Debugging the Deployment  

1. **Apply the manifest** (typically via Helm values that replace the `{{…}}` placeholders):  
   ```bash
   helm upgrade --install ekyc-processor ./move-KYC-and-Esimshub \
     --set dbname=mydb,table=customer_kyc,containerRegistry=myacr.azurecr.io,repository=ekyc-processor,tag=v1.2.3
   ```
   *If using raw `kubectl`:*  
   ```bash
   envsubst < deployment.yml | kubectl apply -f -
   ```

2. **Verify pod status**:  
   ```bash
   kubectl get pods -l app=pgsql-input-processor-ekyc-mydb-customer_kyc
   ```

3. **Inspect logs** (real‑time):  
   ```bash
   kubectl logs -f <pod-name>
   ```

4. **Execute a shell inside the container** (for deeper debugging):  
   ```bash
   kubectl exec -it <pod-name> -- /bin/sh
   # check /app/config, /vaz contents, node version, etc.
   ```

5. **Check health endpoint** (if probes are added later):  
   ```bash
   curl http://<pod-ip>:8081/healthz
   ```

6. **Roll back** (if a bad image/tag is deployed):  
   ```bash
   helm rollback ekyc-processor <previous-revision>
   ```

---

## 7. External Config, Environment Variables & Files  

| Item | Purpose | Source |
|------|---------|--------|
| `NODE_ENV=production` | Forces the Node.js app into production mode (e.g., disables dev logging). | Hard‑coded in manifest. |
| `NODE_CONFIG_DIR=/app/config` | Tells the `node-config` library where to load JSON/YAML config files (DB credentials, API URLs). | Hard‑coded; the actual files must be provided via a ConfigMap or mounted volume (not shown). |
| `{{containerRegistry}}/{{repository}}-{{dbname}}-{{table}}:{{tag}}` | Fully qualified Docker image reference. | Supplied by CI/CD pipeline or Helm values. |
| PVC `vazindia` | Provides Azure storage for temporary or persistent files. | Must be created beforehand (`PersistentVolumeClaim` definition). |
| Potential Secrets (DB user/password, API keys) | Consumed by the Node.js app via `node-config`. | Expected to be injected as files in `/app/config` or via environment variables from a Kubernetes Secret (not defined in this file). |

---

## 8. Suggested Improvements (TODO)

1. **Add health probes** – Define `livenessProbe` and `readinessProbe` (HTTP GET on `/healthz` or a TCP check on port 8081) to enable automatic restart and proper service routing.
2. **Externalize configuration** – Replace the hard‑coded `env` block with a `ConfigMap` and `Secret` volume mounts, allowing per‑environment overrides without rebuilding the image.

---