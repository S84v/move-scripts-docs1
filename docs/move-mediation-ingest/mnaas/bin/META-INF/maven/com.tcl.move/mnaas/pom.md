# Summary
`pom.properties` is a Maven‑generated descriptor bundled with the `mnaas` artifact. It provides immutable metadata (project location, name, groupId, artifactId, version) that build tools, IDEs, and runtime classpath resolvers use to identify the exact source checkout and version of the MOVE‑Mediation Ingest (MNAAS) component in production deployments.

# Key Components
- **groupId / artifactId / version** – Unique Maven coordinates identifying the `mnaas` binary.
- **m2e.projectLocation** – Absolute filesystem path to the Eclipse workspace checkout; used by the Maven‑Eclipse (m2e) connector for source navigation.
- **m2e.projectName** – Logical project identifier within the Eclipse workspace.

# Data Flow
| Source | Destination | Description |
|--------|-------------|-------------|
| Maven build (`mvn package`) | `META-INF/maven/com.tcl.move/mnaas/pom.properties` inside the JAR | Writes static metadata at build time. |
| Ant `build.xml` (classpath assembly) | Runtime classpath resolution | Reads Maven coordinates to locate matching JARs in the repository. |
| IDE (Eclipse m2e) | Source navigation / debugging | Consumes `projectLocation` to map compiled classes back to source files. |
| No external services, DBs, or queues are accessed. |

# Integrations
- **Maven** – Generates the file during the `process-resources` phase.
- **Eclipse m2e** – Parses `m2e.projectLocation`/`m2e.projectName` for workspace linking.
- **Ant build scripts** – Reference Maven coordinates to locate the `mnaas` JAR when constructing compile‑time and runtime classpaths for production pipelines.
- **Deployment scripts** – May validate the `version` field against release tags before promoting to production.

# Operational Risks
- **Stale path**: `m2e.projectLocation` can become invalid after workspace relocation, breaking IDE source mapping. *Mitigation*: Regenerate artifact after any workspace move or exclude the property from production artifacts.
- **Version drift**: Mismatch between `version` in `pom.properties` and the actual deployed artifact can cause classpath conflicts. *Mitigation*: Enforce CI checks that compare artifact version with repository tag.
- **Hard‑coded absolute paths**: Prevents reproducible builds on different machines. *Mitigation*: Strip `m2e.projectLocation` from production builds via Maven filtering.

# Usage
```bash
# Inspect metadata of a deployed JAR
jar xf mnaas-1.0.jar META-INF/maven/com.tcl.move/mnaas/pom.properties
cat META-INF/maven/com.tcl.move/mnaas/pom.properties
```
During debugging in Eclipse, the IDE automatically reads the file to locate source files.

# Configuration
- No runtime environment variables.
- Generated from the Maven project’s `pom.xml`; any changes require a rebuild.
- Ant scripts may reference `${artifact.groupId}`, `${artifact.artifactId}`, `${artifact.version}` derived from this file.

# Improvements
1. **Exclude `m2e.projectLocation` from production artifacts** using Maven’s `resourceFiltering` to avoid absolute‑path leakage.
2. **Add build timestamp and Git commit hash** to `pom.properties` to enable precise traceability of deployed binaries.