# Summary
`pom.properties` is a Maven‑generated resource packaged with the **mnaas** artifact. It provides static metadata (project location, name, groupId, artifactId, version) that runtime components can read to identify the exact build of the MNAAS library deployed in a telecom production move‑mediation pipeline.

# Key Components
- **`m2e.projectLocation`** – Absolute path of the Eclipse project at build time.  
- **`m2e.projectName`** – Maven‑Eclipse project identifier (`mnaas`).  
- **`groupId`** – Maven coordinate (`com.tcl.move`).  
- **`artifactId`** – Maven coordinate (`mnaas`).  
- **`version`** – Semantic version of the deployed JAR (`1.0`).  

These key‑value pairs are accessed via `java.util.Properties` or the Maven `Artifact` API.

# Data Flow
| Phase | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| **Build** | Maven project metadata (pom.xml) | Maven `resources` plugin copies `pom.properties` into `META-INF/maven/com.tcl.move/mnaas/` | Static file bundled in the JAR/WAR |
| **Runtime** | Class‑loader request for `pom.properties` | `Properties.load()` reads the stream | Provides version & build info to logging, health‑check, or compatibility validation modules |
| **Monitoring** | Monitoring scripts or admin tools | Parse file to verify deployed artifact matches expected version | Alerts if version drift occurs |

No external services, databases, or queues are involved.

# Integrations
- **Maven/Eclipse (`m2e`)** – Generates the file during the `process-resources` phase.  
- **MNAAS utilities** – May call `VersionInfo.get()` (or similar) to load these properties for logging, audit trails, or conditional feature toggles.  
- **Deployment automation** – CI/CD pipelines can read the file post‑deployment to confirm the correct artifact version is on target nodes.  

# Operational Risks
- **Stale metadata** – If the file is not regenerated after a code change, version reporting becomes inaccurate. *Mitigation*: enforce clean rebuilds; fail the build if `pom.properties` timestamp is older than source changes.  
- **Hard‑coded absolute path** (`m2e.projectLocation`) may expose internal filesystem structure. *Mitigation*: strip or mask this entry in production builds via Maven filtering.  
- **Missing file** (e.g., excluded from packaging). *Mitigation*: add a Maven `verify` check to ensure `pom.properties` exists in the final artifact.

# Usage
```java
// Example: retrieve MNAAS version at runtime
Properties p = new Properties();
try (InputStream is = getClass()
        .getResourceAsStream("/META-INF/maven/com.tcl.move/mnaas/pom.properties")) {
    p.load(is);
}
String version = p.getProperty("version");   // "1.0"
String artifact = p.getProperty("artifactId"); // "mnaas"
System.out.println("Running MNAAS version " + version);
```
Command‑line verification:
```bash
jar tf mnaas-1.0.jar | grep pom.properties
jar xf mnaas-1.0.jar META-INF/maven/com.tcl.move/mnaas/pom.properties
cat META-INF/maven/com.tcl.move/mnaas/pom.properties
```

# Configuration
- No environment variables are consumed.  
- The file itself is the sole configuration artifact referenced by runtime code that needs build metadata.

# Improvements
1. **Add build timestamp and Git commit hash** to `pom.properties` via Maven `buildnumber-maven-plugin` to improve traceability.  
2. **Exclude `m2e.projectLocation` from production artifacts** using Maven resource filtering to avoid leaking internal paths.