**File:** `move-Inventory\move-BOSS-Maximity\azure-pipelines.yml`  

---

## 1. High‑Level Summary
This Azure DevOps pipeline definition packages the entire source tree of the *Maximity Input Processor* (the “move‑BOSS‑Maximity” component) into a zip archive and publishes it as a build artifact named **maximity_input_processor**. It is intended to be run on demand (no CI trigger) and supplies the artifact for downstream pipelines that will build Docker images, deploy to Azure Container Registry, or run integration tests.

---

## 2. Pipeline Structure (Key “Classes/Functions”)

| Element | Type | Responsibility |
|---------|------|----------------|
| **variables** | pipeline‑level variables | Holds reusable values: `dockerfilePath` (path to Dockerfile), `tag` (image tag placeholder), `vmImageName` (agent image). |
| **stage: PublishArtifact** | Stage | Groups all steps required to create and publish the zip artifact. |
| **job: Copy** | Job | Executes on the selected agent; performs the actual archiving and publishing. |
| **task: ArchiveFiles@2** | Task | Creates `maximity_input_processor.zip` from the root of the repository, preserving the folder hierarchy. |
| **task: CopyFiles@2** | Task | Copies the generated zip (and any other `*.zip` files) to the staging folder for publishing. |
| **task: PublishBuildArtifacts@1** | Task | Uploads the staging folder contents to Azure DevOps as a named artifact (`maximity_input_processor`). |

*No custom scripts, classes, or functions are defined in this YAML; the pipeline relies entirely on built‑in Azure DevOps tasks.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • Repository source (`$(Build.SourcesDirectory)`) – the full code checkout.<br>• Implicit pipeline variables (`Build.SourcesDirectory`, `Build.ArtifactStagingDirectory`). |
| **Outputs** | • `maximity_input_processor.zip` (artifact).<br>• Azure DevOps artifact `maximity_input_processor` stored in the build’s artifact container. |
| **Side Effects** | • Consumes compute resources on an `ubuntu-latest` hosted agent.<br>• Writes a zip file to the agent’s temporary disk.<br>• Uploads the artifact to Azure DevOps storage (network I/O). |
| **Assumptions** | • The repository contains a valid Dockerfile at the path referenced by `dockerfilePath` (currently unused).<br>• The build agent has sufficient disk space for the full source tree + zip.<br>• No CI trigger is required; the pipeline will be started manually or by another pipeline. |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | How the Connection Occurs |
|---------------------|----------------------------|
| **Downstream Docker Build Pipeline** (e.g., `move-Inventory\move-BOSS-Maximity\docker-build.yml`) | Consumes the `maximity_input_processor` artifact, extracts the source, and runs `docker build` using the `dockerfilePath` variable. |
| **Deployment/Release Pipelines** | Pull the published artifact to stage the code on test or production environments, possibly feeding it to a container registry or a runtime orchestrator (Kubernetes, Azure Container Apps). |
| **Configuration Management** | May reference the same variable group that defines `dockerfilePath`, `tag`, and `vmImageName` for consistency across pipelines. |
| **Monitoring / Auditing** | Azure DevOps build history records the artifact version (`maximity_input_processor_YYYYMMDDr`) which downstream pipelines can reference for traceability. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Artifact Size Blow‑up** – Large repo or binary files cause the zip to exceed Azure DevOps artifact limits (2 GB). | Build fails or slows down; downstream pipelines may time‑out. | Add a `.artifactignore` (or adjust `ArchiveFiles` `includeRootFolder`/`exclude` patterns) to omit unnecessary large files (e.g., `node_modules`, `*.log`). |
| **Missing Files** – Critical files (Dockerfile, config) not included because of path errors. | Downstream Docker build fails. | Verify the archive contains expected paths (`tree` step or `PublishBuildArtifacts` log). |
| **Agent Resource Exhaustion** – Insufficient disk or memory on the hosted agent. | Build aborts. | Use a self‑hosted agent with larger resources for very large codebases, or split the archive into multiple parts. |
| **Stale Artifact Naming** – The artifact name is static; concurrent runs may overwrite each other. | Loss of previous build artifacts. | Include a unique build number or commit SHA in the artifact name (`maximity_input_processor_$(Build.BuildId).zip`). |
| **Unused Variables** – `dockerfilePath`, `tag`, `vmImageName` are defined but not used, potentially causing confusion. | Maintenance overhead. | Clean up unused variables or document their intended future use. |

---

## 6. Running / Debugging the Pipeline

1. **Trigger Manually**  
   - In Azure DevOps UI, navigate to *Pipelines → <pipeline name>* → *Run pipeline*.  
   - Optionally set a custom run name or override variables (e.g., change `tag`).  

2. **Monitor Execution**  
   - Click the running job to view real‑time logs for each task (`ArchiveFiles`, `CopyFiles`, `PublishBuildArtifacts`).  
   - Verify the `ArchiveFiles` step reports the number of files archived and the final zip size.  

3. **Validate Artifact**  
   - After success, go to *Runs → <run> → Artifacts* and download `maximity_input_processor`.  
   - Inspect the zip locally (`unzip -l`) to confirm required files are present.  

4. **Debug Failures**  
   - If `ArchiveFiles` fails, check the `rootFolderOrFile` path and any exclusion patterns.  
   - If `PublishBuildArtifacts` fails, verify the service connection has write permission to the artifact container and that the staging folder exists.  

5. **Rerun**  
   - Use the *Run pipeline* UI to re‑run with the same or modified variables.  
   - For automated retries, add a `retry` policy to the job (not currently present).  

---

## 7. External Configuration, Environment Variables & Files

| Item | Usage |
|------|-------|
| **`dockerfilePath`** (pipeline variable) | Currently unused; intended for downstream Docker build pipelines. |
| **`tag`** (pipeline variable) | Placeholder for Docker image tag; not referenced in this file. |
| **`vmImageName`** (pipeline variable) | Determines the hosted agent image (`ubuntu-latest`). |
| **Azure DevOps Service Connection** (implicit) | Required for `PublishBuildArtifacts` to write to the artifact container. |
| **`.gitignore` / `.artifactignore`** (potential) | Not referenced here but would control what gets archived. |

If the repository contains a `Dockerfile` or other config files, they are simply packaged as part of the zip; no special handling is performed.

---

## 8. Suggested Improvements (TODO)

1. **Add a Docker Build Stage** – Extend the pipeline to build the Docker image using the `dockerfilePath` and `tag` variables, then push to the Azure Container Registry. This would make the pipeline end‑to‑end for the *maximity_input_processor* component.  
2. **Parameterize Artifact Naming** – Include `$(Build.BuildId)` or the commit SHA in the zip filename and artifact name to avoid overwriting artifacts from concurrent runs and to improve traceability.  

---