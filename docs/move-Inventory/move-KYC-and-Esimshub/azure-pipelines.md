**File:** `move-Inventory\move-KYC-and-Esimshub\azure-pipelines.yml`  

---

## 1. High‑Level Summary
This Azure DevOps pipeline definition packages the *KYC‑and‑Esimshub* move‑script repository into a zip artifact (`pgsql_input_processor.zip`) and publishes it to the pipeline’s artifact container. It is a **manual‑triggered** (no CI trigger) build that prepares the source for downstream consumption (e.g., Docker image build, deployment, or manual download). Although Docker‑related variables are declared, the current YAML only performs artifact creation; no container build or push occurs.

---

## 2. Key Sections & Responsibilities  

| Section / Element | Responsibility |
|-------------------|----------------|
| `trigger: none` | Disables automatic runs on repo changes; pipeline must be queued manually or via another pipeline. |
| `variables` | Declares reusable values: `dockerfilePath`, `tag`, `vmImageName`. Currently unused but indicate future Docker build intent. |
| `name: pgsql_input_processor_$(Date:yyyyMMdd)$(Rev:.r)` | Generates a unique run name based on the date and a revision counter. |
| **Stage – `PublishArtifact`** | Collects the repository contents, creates a zip, and publishes it as a build artifact. |
| **Job – `Copy`** | Executes three tasks: |
| `ArchiveFiles@2` | Zips the entire source directory (`$(Build.SourcesDirectory)`) into `pgsql_input_processor.zip`. |
| `CopyFiles@2` | Copies the generated zip (and any other `*.zip` files) to the staging folder (`$(Build.ArtifactStagingDirectory)`). |
| `PublishBuildArtifacts@1` | Publishes the staging folder as an artifact named `pgsql_input_processor` to Azure DevOps artifact storage. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Repository source (`$(Build.SourcesDirectory)`). <br> - Implicit environment variables supplied by Azure Pipelines (e.g., `Build.SourcesDirectory`, `Build.ArtifactStagingDirectory`). |
| **Outputs** | - Artifact `pgsql_input_processor` containing `pgsql_input_processor.zip`. |
| **Side Effects** | - Stores the artifact in Azure DevOps (available for download or downstream pipelines). <br> - Consumes build agent resources (Ubuntu‑latest VM). |
| **Assumptions** | - The repository contains a valid `Dockerfile` at the path referenced by `dockerfilePath` (future use). <br> - The build agent has permission to write to the staging directory and publish artifacts. <br> - No large binary files that would cause the zip to exceed Azure DevOps size limits (currently 2 GB). |
| **External Services** | - Azure DevOps Pipelines service (agent pool, artifact storage). <br> - (Potential) Azure Container Registry – referenced only by comment/variable, not used in this version. |

---

## 4. Integration Points with the Rest of the System  

| Connected Component | How This Pipeline Links |
|---------------------|--------------------------|
| **Other Move Scripts (e.g., `move-Inventory\move-BOSS-Maximity\*`)** | The produced artifact can be consumed by downstream pipelines that build Docker images for the *BOSS‑Maximity* or *KYC‑Esimshub* moves. Those pipelines likely reference the same repository and expect a zip payload. |
| **Docker Build / Deploy Pipelines** | The declared variables (`dockerfilePath`, `tag`, `vmImageName`) suggest a later stage (or separate pipeline) will pull this artifact, run `docker build` using the Dockerfile, tag the image (`vaz1`), and push to an Azure Container Registry. |
| **Release Pipelines** | Release definitions that deploy the KYC/Esimshub service will retrieve the `pgsql_input_processor` artifact as the source code bundle. |
| **Manual Operations** | Operators can download the zip directly from the pipeline run UI for ad‑hoc testing or hot‑fixes. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Artifact Size Blow‑up** – Adding large binaries or logs could exceed the 2 GB artifact limit, causing pipeline failure. | Build fails; downstream processes cannot retrieve code. | Enforce `.gitignore`/`.npmignore` rules; add a `clean` step to prune unnecessary files before archiving. |
| **Stale Docker Variables** – `dockerfilePath`, `tag`, `vmImageName` are defined but unused, potentially confusing maintainers. | Misinterpretation of pipeline purpose; accidental reliance on missing steps. | Remove unused variables or add a comment indicating they are placeholders for future stages. |
| **Manual Trigger Dependency** – `trigger: none` means the artifact is not refreshed automatically after code changes. | Out‑of‑date artifact used in production; human error in queuing. | Add a scheduled trigger or integrate this pipeline as a downstream job of a CI build that runs on each commit. |
| **Missing Dockerfile** – If a downstream Docker build expects the Dockerfile at the path defined, a missing file will cause later failures. | Deployment breakage. | Add a validation step (`CheckFileExists`) before publishing the artifact, or document the required file location clearly. |
| **Agent Compatibility** – The pipeline assumes `ubuntu-latest` has required tools (zip, Azure CLI). | Unexpected task failures on agent updates. | Pin the VM image version (e.g., `ubuntu-22.04`) and include a task to verify required tools are present. |

---

## 6. Running / Debugging the Pipeline  

1. **Queue Manually**  
   - In Azure DevOps, navigate to *Pipelines → <pipeline name>* and click **Run pipeline**.  
   - Optionally override variables (e.g., change `tag`) if needed for testing.  

2. **Monitor Execution**  
   - Watch the *Copy* job logs. Key sections:  
     - `ArchiveFiles` – confirms the zip file name and size.  
     - `CopyFiles` – shows source and destination paths.  
     - `PublishBuildArtifacts` – displays the artifact name and URL.  

3. **Validate Output**  
   - After success, go to the *Artifacts* tab of the run, download `pgsql_input_processor.zip`, and inspect its contents.  

4. **Debug Common Issues**  
   - **Missing Files** – If the zip is empty, verify `$(Build.SourcesDirectory)` is correctly populated (checkout step may have been disabled).  
   - **Permission Errors** – Ensure the build service account has *Contribute* rights to the artifact container.  
   - **Agent Failures** – Look for “agent not found” or “task not recognized” messages; update the pipeline’s `vmImageName` or install missing extensions.  

---

## 7. External Config / Environment References  

| Reference | Usage |
|-----------|-------|
| `$(Build.SourcesDirectory)` | Root of the repository checkout; source for archiving. |
| `$(Build.ArtifactStagingDirectory)` | Temporary staging folder where the zip is placed before publishing. |
| `dockerfilePath`, `tag`, `vmImageName` | Declared but not used; intended for future Docker build steps. |
| Azure DevOps Service Connection (implied) | Required for the `PublishBuildArtifacts` task to store artifacts. |

---

## 8. Suggested Improvements (TODO)

1. **Add Docker Build & Push Stage** – Leverage the existing variables to build the Docker image from `Dockerfile` and push it to the Azure Container Registry, making the pipeline end‑to‑end.  
2. **Introduce a Validation Step** – Before archiving, run a script that checks for the presence of required files (e.g., `Dockerfile`, configuration files) and fails fast with a clear error message if they are missing.  

---