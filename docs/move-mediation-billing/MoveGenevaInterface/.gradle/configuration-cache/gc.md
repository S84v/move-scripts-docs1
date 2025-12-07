# Summary
`move-mediation-billing\MoveGenevaInterface\.gradle\configuration-cache\gc.properties` is a Gradle‑generated properties file that records the JVM garbage‑collection (GC) settings used by the Gradle daemon when the **configuration‑cache** feature is active. In production it ensures that successive builds of the *MoveGenevaInterface* module reuse a daemon with identical GC parameters, guaranteeing deterministic memory‑management behavior and preventing configuration‑cache invalidation caused by mismatched JVM flags.

# Key Components
- **gc.heapSize** – Maximum heap size allocated to the Gradle daemon (e.g., `-Xmx2g`).
- **gc.gcAlgorithm** – Selected GC algorithm (e.g., `-XX:+UseG1GC`).
- **gc.logFile** – Path to the GC log file for diagnostic purposes.
- **gc.flags** – Additional JVM arguments (e.g., `-XX:+HeapDumpOnOutOfMemoryError`).

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input** | Read by Gradle Wrapper (`gradlew`) at daemon start | Parses `gc.properties` to construct the JVM command line for the daemon. |
| **Output** | Written by Gradle after daemon launch or when GC settings change | Updated `gc.properties` reflecting the active JVM arguments. |
| **Side Effects** | Influences daemon memory footprint and GC logging | Affects build performance and stability of the *MoveGenevaInterface* pipeline. |
| **External Services** | None directly; indirect impact on Spark jobs launched by the module (memory pressure). |
| **Databases / Queues** | Not applicable. |

# Integrations
- **Gradle Wrapper (`gradlew`)** – Consumes the file to configure the daemon.
- **Configuration‑Cache** – Requires identical JVM flags across builds; the file is part of the cache validation logic.
- **CI/CD pipelines** – Build agents reference the file to ensure consistent daemon reuse across stages.
- **Monitoring tools** – GC log path may be ingested by log aggregation (e.g., ELK) for performance analysis.

# Operational Risks
- **Stale or mismatched GC settings** – May cause daemon restart, leading to longer build times. *Mitigation*: enforce version‑controlled Gradle wrapper and periodic cleanup of `.gradle` directories.
- **Excessive heap allocation** – Could starve host resources on shared build agents. *Mitigation*: cap `gc.heapSize` based on agent specifications.
- **Missing GC log file** – Loss of diagnostic data. *Mitigation*: ensure log directory exists and is writable.

# Usage
```bash
# Trigger a build that respects the stored GC settings
./gradlew :MoveGenevaInterface:clean build

# Debug: view the effective JVM args used by the daemon
./gradlew --status | grep "JVM arguments"
```
To manually adjust settings, edit `gc.properties` and restart the daemon:
```bash
./gradlew --stop   # stop existing daemon
# edit gc.properties as needed
./gradlew build    # daemon restarts with new GC args
```

# configuration
- **Environment Variables**  
  - `GRADLE_OPTS` – Overrides values in `gc.properties` if set.  
  - `JAVA_HOME` – Determines the JDK used by the daemon.
- **Referenced Config Files**  
  - `gradle.properties` (project‑wide defaults).  
  - `settings.gradle` (enables configuration‑cache via `org.gradle.configuration-cache=true`).

# Improvements
1. **Automated Validation** – Add a Gradle task (`validateGcProperties`) that checks heap size against the build agent’s limits and fails early if out‑of‑bounds.  
2. **Centralised Management** – Store canonical GC settings in a version‑controlled `gradle/gc-defaults.properties` and have a pre‑build script copy them into each module’s `.gradle/configuration-cache/gc.properties` to guarantee consistency across all modules.