# Summary
`pom.properties` is a Maven‑generated resource packaged with the **mnaas** artifact. It supplies immutable build metadata (project location, name, groupId, artifactId, version) that runtime components of the Move‑Mediation‑Ingest (MNAAS) pipeline read to identify the exact library version deployed in production.

# Key Components
- **`groupId`** – Maven coordinate identifying the organization (`com.tcl.move`).
- **`artifactId`** – Maven coordinate identifying the module (`mnaas`).
- **`version`** – Semantic version of the deployed artifact (`1.0`).
- **`m2e.projectLocation`** – Absolute filesystem path of the Eclipse project at build time (used for debugging or source‑level diagnostics).
- **`m2e.projectName`** – Eclipse project identifier (`mnaas`).

# Data Flow
- **Input:** None (static file generated at build time).
- **Output:** In‑memory key‑value map consumed by:
  - MNAAS job launch scripts (`java -cp … com.tcl.move.mnaas.*`) to verify version compatibility.
  - Monitoring agents that report artifact version to telemetry dashboards.
- **Side Effects:** None.
- **External Services/DBs/Queues:** None.

# Integrations
- **Hadoop Configuration Builder:** Reads `groupId`, `artifactId`, `version` to construct class‑path entries for `-libjars`.
- **Deployment Automation (e.g., Jenkins, GitLab CI):** Publishes the artifact; the generated `pom.properties` is included in the JAR/WAR for downstream validation.
- **Runtime Diagnostics:** Logging utilities query this file to embed version info in log headers.

# Operational Risks
- **Stale Metadata:** If the file is not regenerated after a code change, version mismatch may cause silent incompatibilities. *Mitigation:* Enforce Maven clean‑install before each release; validate checksum in CI.
- **Hard‑coded Path Exposure:** `m2e.projectLocation` reveals internal build paths, potentially leaking infrastructure details. *Mitigation:* Strip or mask this property in production builds via Maven filtering.

# Usage
```bash
# Retrieve version at runtime (Java example)
InputStream is = Thread.currentThread()
    .getContextClassLoader()
    .getResourceAsStream("META-INF/maven/com.tcl.move/mnaas/pom.properties");
Properties p = new Properties();
p.load(is);
String version = p.getProperty("version");
System.out.println("MNAAS version: " + version);
```

# Configuration
- No environment variables required.
- File location is fixed inside the JAR at `META-INF/maven/com.tcl.move/mnaas/pom.properties`.
- Maven `pom.xml` controls generation via the `maven-resources-plugin`.

# Improvements
1. **Remove Development‑Only Fields:** Exclude `m2e.projectLocation` and `m2e.projectName` from production artifacts using Maven resource filtering.
2. **Add SHA‑256 Digest:** Include a checksum field (`artifactSha256`) to enable integrity verification during deployment.