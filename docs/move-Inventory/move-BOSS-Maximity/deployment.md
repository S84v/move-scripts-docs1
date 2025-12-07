**File:** `move-Inventory\move-BOSS-Maximity\deployment.yml`  

---

## 1. High‑Level Summary
This Kubernetes **Deployment** manifest defines the runtime environment for the *Maximity Input Processor* – a Node.js service (`app.js`) that moves, transforms, and orchestrates inventory data between the BOSS and Maximity systems. For each logical data set (identified by `{{dbname}}` and `{{table}}`) a dedicated pod is created, pulling the appropriate container image, exposing port 8081, and mounting a persistent Azure volume (`vazindia`) that holds shared assets (e.g., SFTP keys, staging files). The manifest is templated; CI/CD pipelines substitute the placeholders with concrete values before applying to the cluster.

---

## 2. Important Sections & Their Responsibilities  

| Section | Responsibility |
|---------|----------------|
| `metadata.name` / `metadata.labels` | Unique identifier for the Deployment; ties pods to the specific database/table pair. |
| `spec.replicas: 1` | Guarantees a single instance (single‑threaded processing) per data set. |
| `selector.matchLabels` | Ensures the Deployment controls only pods with the matching `app` label. |
| `template.spec.containers[0]` | Defines the container runtime: image, command, environment, resources, ports, and volume mounts. |
| `env` block | Sets `NODE_ENV=production` and points the Node.js config loader to `/app/config`. |
| `resources.requests.cpu: 70m` | Minimal CPU reservation; relies on cluster autoscaling for burst capacity. |
| `image` (templated) | Pulls the exact version (`{{tag}}`) of the service built for the given `{{dbname}}`/`{{table}}`. |
| `command: ["node","app.js"]` | Starts the Node.js entry point that implements the move logic (see `app.js`). |
| `ports.containerPort: 8081` | Exposes the internal HTTP/metrics endpoint (used by health checks or monitoring). |
| `volumeMounts` / `volumes` | Mounts the Azure Persistent Volume Claim `vazindia` at `/vaz` inside the container, providing shared storage for input files, logs, or external credentials. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Template Inputs** | `{{dbname}}`, `{{table}}`, `{{containerRegistry}}`, `{{repository}}`, `{{tag}}` – supplied by the CI pipeline (`azure-pipelines.yml`). |
| **External Resources** | - Azure PVC `vazindia` (must exist in the same namespace).<br>- Config directory `/app/config` (populated via a ConfigMap or mounted volume not shown here). |
| **Outputs** | - A `Deployment` object in the target namespace.<br>- One running `Pod` named `maximity-input-processor-…` that executes `app.js`. |
| **Side Effects** | - Pulls the container image from the specified registry (requires registry credentials).<br>- Mounts the PVC, potentially reading/writing large files.<br>- Opens TCP 8081 (used by internal monitoring). |
| **Assumptions** | - The cluster has access to the container registry (imagePullSecrets configured elsewhere).<br>- The PVC `vazindia` is bound to an Azure Disk/Files share with sufficient capacity.<br>- `app.js` expects configuration files under `/app/config` and data under `/vaz`. |
| **Dependencies** | - `move-Inventory\move-BOSS-Maximity\app.js` (the Node.js process).<br>- CI pipeline (`azure-pipelines.yml`) that renders the template and applies it. |

---

## 4. Connection to Other Scripts & Components  

| Component | Relationship |
|-----------|--------------|
| **`app.js`** (Node entry point) | Executed by the container; reads configuration, processes files from `/vaz`, writes results back, and may call external APIs (BOSS, Maximity). |
| **`azure-pipelines.yml`** | Supplies the templating values (`dbname`, `table`, `tag`, etc.) and runs `kubectl apply -f deployment.yml`. |
| **ConfigMap / Secret (not shown)** | Likely provides the `/app/config` directory (e.g., API keys, endpoint URLs). |
| **Other Deployments** | May run in parallel for different `dbname/table` combos; they share the same PVC (`vazindia`) for cross‑job coordination. |
| **Monitoring / Alerting** | Port 8081 can be scraped by Prometheus or health‑checked by Kubernetes liveness/readiness probes (not defined here). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or mis‑bound PVC** | Pod fails to start (mount error) → data‑move stops. | Validate PVC existence with `kubectl get pvc vazindia` before deployment; add an `initContainer` that checks mount accessibility. |
| **Incorrect templating values** (e.g., wrong `dbname`) | Deploys to wrong dataset, causing data corruption. | Enforce CI validation (lint the rendered YAML) and gate merges with a pre‑deployment script that cross‑checks against a master table of allowed values. |
| **Image pull failures** (registry auth, tag typo) | Pod stays in `ImagePullBackOff`. | Ensure imagePullSecrets are defined at namespace level; use immutable tags and verify existence via `docker manifest inspect`. |
| **Insufficient CPU request** (70 m) | Pod may be throttled under load, leading to timeouts. | Add a `resources.limits` block and monitor CPU usage; adjust request based on observed load. |
| **No health probes** | Kubernetes cannot detect hung processes, leading to prolonged outages. | Add `livenessProbe` (e.g., HTTP GET `/healthz` on port 8081) and `readinessProbe`. |
| **Unbounded log growth** (writes to PVC) | Disk exhaustion on the Azure share. | Rotate logs inside the container or configure a side‑car log shipper; set PVC size limits. |

---

## 6. Running / Debugging the Deployment  

1. **Render the template** (CI step or locally):  
   ```bash
   envsubst < deployment.yml > rendered.yml
   ```
   (Replace placeholders with real values.)

2. **Apply to the cluster**:  
   ```bash
   kubectl apply -f rendered.yml
   ```

3. **Verify creation**:  
   ```bash
   kubectl get deployment -l app=maximity-input-processor-<dbname>-<table>
   kubectl get pod -l app=maximity-input-processor-<dbname>-<table>
   ```

4. **Inspect logs** (primary debugging entry point):  
   ```bash
   POD=$(kubectl get pod -l app=maximity-input-processor-<dbname>-<table> -o jsonpath='{.items[0].metadata.name}')
   kubectl logs $POD
   ```

5. **Check PVC mount**:  
   ```bash
   kubectl exec $POD -- df -h /vaz
   ```

6. **Port‑forward for local health check** (if `/healthz` endpoint exists):  
   ```bash
   kubectl port-forward $POD 8081:8081
   curl http://localhost:8081/healthz
   ```

7. **Roll back** (if needed):  
   ```bash
   kubectl rollout undo deployment/maximity-input-processor-<dbname>-<table>
   ```

---

## 7. External Config / Environment Variables  

| Variable | Source | Usage |
|----------|--------|-------|
| `NODE_ENV=production` | Hard‑coded in manifest | Informs Node.js to load production config (`config/production.json`). |
| `NODE_CONFIG_DIR=/app/config` | Hard‑coded | Directs the `config` npm module to read JSON/YAML files from the container’s `/app/config` path (likely mounted via ConfigMap). |
| `{{containerRegistry}}/{{repository}}-{{dbname}}-{{table}}:{{tag}}` | Supplied by CI pipeline (`azure-pipelines.yml`) | Fully qualified image reference. |
| `{{dbname}}`, `{{table}}` | Supplied by CI (derived from job parameters) | Drives naming, image selection, and potentially runtime logic inside `app.js`. |
| `vazindia` PVC | Cluster resource (created elsewhere) | Provides persistent storage for input files, staging data, and possibly credentials. |

---

## 8. Suggested TODO / Improvements  

1. **Add health probes** – Define `livenessProbe` and `readinessProbe` (HTTP GET `/healthz` on port 8081) to enable automatic restarts and proper service discovery.  
2. **Specify resource limits** – Complement the current CPU request with a `limits` block (e.g., `cpu: 200m`, `memory: 256Mi`) and add a memory request to prevent OOM kills.  

---