**File:** `move-mediation-ingest/azure-pipelines.yml`  

---

## 1. High‑Level Summary
This Azure DevOps pipeline definition automates the continuous‑integration (CI) process for the **MNAAS** Java component of the mediation‑ingest stack. On every push to the `master` branch it provisions an Ubuntu build agent, runs a Maven build (`package` goal) using JDK 1.8, executes unit tests, and publishes the JUnit test results. The resulting artifact (typically a JAR/WAR) is left on the build agent for downstream steps (e.g., release pipelines or manual download).

---

## 2. Key Sections & Responsibilities  

| Section | Responsibility |
|---------|----------------|
| `trigger` | Starts the pipeline automatically for any commit to the `master` branch. |
| `pool` | Selects the hosted Ubuntu‑latest build agent (`ubuntu-latest`). |
| `steps` → `Maven@3` | Executes Maven with the supplied `pom.xml`, allocates up to 3 GB heap, forces JDK 1.8 (x64), runs unit tests, and publishes JUnit XML results. |
| `inputs` (inside Maven task) | Configures the Maven command: <br>• `mavenPomFile` – location of the project descriptor (`mnaas/pom.xml`). <br>• `mavenOptions` – JVM options for Maven (`-Xmx3072m`). <br>• `javaHomeOption`, `jdkVersionOption`, `jdkArchitectureOption` – enforce JDK 1.8, 64‑bit. <br>• `publishJUnitResults` – true, so Azure DevOps records test outcomes. <br>• `testResultsFiles` – glob pattern for locating JUnit XML files. <br>• `goals` – Maven lifecycle phase (`package`). |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Source code repository (the pipeline runs in the same repo). <br>• Maven `pom.xml` located at `mnaas/pom.xml`. <br>• Any Maven settings files (e.g., `settings.xml`) that may be referenced via environment variables or the agent’s global Maven config. |
| **Outputs** | • Build artifact(s) produced by `mvn package` (e.g., `target/*.jar` or `*.war`). <br>• JUnit test result XML files published to Azure DevOps test tab. |
| **Side Effects** | • Consumes build agent resources (CPU, memory, disk). <br>• May download Maven dependencies from external repositories (e.g., Maven Central, corporate Nexus/Artifactory). |
| **Assumptions** | • The repository contains a valid Maven project under `mnaas/`. <br>• All required Maven dependencies are reachable from the build agent (no firewall blocks). <br>• JDK 1.8 is compatible with the codebase. <br>• Test reports are generated under `**/surefire-reports/TEST-*.xml`. |

---

## 4. Integration Points with the Rest of the System  

| Connection | Description |
|------------|-------------|
| **Downstream Release Pipelines** | The artifact produced here is typically consumed by a release pipeline (e.g., `azure-pipelines-release.yml`) that deploys the JAR/WAR to a staging or production environment (Docker, Kubernetes, or on‑prem servers). |
| **Shared Maven Settings** | If the organization uses a corporate Maven repository, a `settings.xml` may be injected via a variable group or a service connection; this pipeline will rely on that configuration. |
| **Telemetry / Monitoring** | Test results and build status feed into Azure DevOps dashboards that are referenced by the broader “move‑mediation‑ingest” monitoring suite. |
| **Versioning Scripts** | Other scripts (e.g., `utils/version.js` in the move‑MSPS stack) may read the generated artifact’s version from the `pom.xml` or from the built JAR’s manifest for downstream data‑move jobs. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Build Agent Out‑of‑Memory** – Maven may exceed the 3 GB heap on large projects. | Increase `mavenOptions` (`-Xmx4g`) or enable Maven’s parallel builds (`-T 1C`). |
| **Dependency Resolution Failures** – External repositories unavailable. | Cache Maven dependencies using Azure Pipelines caching task; add a fallback repository URL. |
| **JDK Version Drift** – Future JDK upgrades could break compilation. | Pin the JDK version via a pipeline variable (`JAVA_VERSION=1.8`) and lock the hosted image version (`ubuntu-20.04`). |
| **Test Flakiness** – Intermittent test failures cause pipeline failures. | Add a retry step for flaky tests or separate stable vs. integration test suites. |
| **Missing Artifact Publication** – Downstream pipelines cannot locate the built JAR. | Add an explicit `PublishBuildArtifacts` step to push `target/*.jar` to the pipeline’s artifact store. |

---

## 6. Running & Debugging the Pipeline  

1. **Trigger** – Push a commit to `master`; Azure DevOps automatically queues the pipeline.  
2. **Manual Run** – In Azure DevOps UI, navigate to *Pipelines → azure-pipelines.yml* and click **Run pipeline**.  
3. **Viewing Logs** – Click the running job, expand the **Maven** step to see the full `mvn` command, console output, and test summary.  
4. **Debugging Failures** –  
   * Check the **Tests** tab for detailed JUnit failures.  
   * Re‑run the pipeline with **Enable system diagnostics** (pipeline variable `system.debug = true`) to get verbose Maven logs.  
   * If dependency resolution fails, inspect the agent’s `~/.m2/repository` cache or add a `Maven@3` step with `options: '-X'` for debug output.  
5. **Artifact Retrieval** – After a successful run, download the built JAR from the **Summary → Artifacts** section (once a `PublishBuildArtifacts` step is added).  

---

## 7. External Configuration & Environment Variables  

| Variable / File | Usage |
|-----------------|-------|
| `JAVA_HOME` (set by `javaHomeOption`) | Determines which JDK the Maven task uses. |
| `MAVEN_OPTS` (derived from `mavenOptions`) | Passes JVM heap size to Maven. |
| `system.debug` (optional) | Enables detailed logging when set to `true`. |
| Potential **service connections** (e.g., Maven repository credentials) | Not shown in this file but may be referenced via pipeline variable groups. |

---

## 8. Suggested Improvements (TODO)

1. **Publish Build Artifacts** – Add a `PublishBuildArtifacts@1` step to store the generated JAR/WAR for downstream consumption.  
2. **Cache Maven Dependencies** – Insert a `Cache@2` task keyed on `pom.xml` checksum to speed up builds and reduce external repository load.  

---