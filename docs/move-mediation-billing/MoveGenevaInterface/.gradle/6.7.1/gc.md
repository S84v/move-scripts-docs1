# Summary
`gc.properties` is a Gradle‑generated properties file located under the version‑specific cache directory (`.gradle/6.7.1`). It stores the JVM arguments that the Gradle daemon uses for garbage‑collection tuning (e.g., heap size, GC algorithm, log settings). In production, the file is read by the Gradle Wrapper (`gradlew.bat` / `gradlew`) each time the daemon is started for the **MoveGenevaInterface** module, ensuring consistent memory‑management behavior across builds, tests, and packaging tasks.

# Key Components
- **`org.gradle.jvmargs`** – Comma‑separated list of JVM options applied to the daemon (heap limits, GC flags, `-Dfile.encoding=UTF‑8`, etc.).
- **`org.gradle.daemon.idletimeout`** – Idle timeout (in seconds) after which the daemon shuts down; influences GC pressure on the host.
- **`org.gradle.logging.level`** – Controls verbosity of daemon logs, including GC diagnostics if enabled.
- **Generated comment header** – Indicates Gradle version (`6.7.1`) and timestamp of creation.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | Gradle Wrapper execution (`gradlew.bat`) | Wrapper resolves Java, reads `gradle-wrapper.properties`, then loads `gc.properties` from the user’s `.gradle/6.7.1` cache. | JVM arguments are passed to the daemon JVM. |
| 2 | Daemon start | JVM is launched with the arguments from `org.gradle.jvmargs`. | Daemon process with configured GC runs; may emit GC logs to the location defined in the properties. |
| 3 | Build tasks (compile, test, package) | Daemon executes tasks for **MoveGenevaInterface**. | Build artifacts (JAR, Docker image, etc.) are produced; memory usage is governed by the GC settings. |
| 4 | Daemon shutdown (idle timeout) | Daemon terminates after `org.gradle.daemon.idletimeout`. | Resources released; GC logs may be flushed. |

External services touched indirectly:
- **Java runtime** (resolved via `JAVA_HOME` or PATH)
- **File system** (writes build outputs, optional GC logs)
- **Gradle Wrapper** (bootstrap script)

# Integrations
- **`gradlew.bat` / `gradlew`** – Reads `gc.properties` when constructing the daemon command line.
- **`MoveGenevaInterface` Gradle build scripts** – Inherit the daemon JVM args; any custom `org.gradle.jvmargs` defined in `build.gradle` are merged with those from `gc.properties`.
- **CI/CD pipelines** – May inject additional JVM args via `GRADLE_OPTS`; these override or augment the values in `gc.properties`.
- **IDE integrations (IntelliJ/Eclipse)** – When invoking Gradle tasks from the IDE, the same daemon configuration is applied.

# Operational Risks
- **Out‑of‑Memory (OOM) crashes** – If `-Xmx` is set too low for the module’s Spark/Hive dependencies, the daemon may terminate mid‑build. *Mitigation*: Align heap size with `MoveGenevaInterface` memory profile; monitor daemon logs.
- **Excessive GC pause times** – Inappropriate GC algorithm (e.g., using ParallelGC on a latency‑sensitive build) can increase build latency. *Mitigation*: Use G1GC or ZGC for large heaps; enable GC logging to tune.
- **Stale configuration** – The file is cached per Gradle version; after a Gradle upgrade, old settings may persist, causing unexpected behavior. *Mitigation*: Delete the `.gradle/6.7.1` directory when upgrading or explicitly regenerate the file.
- **Security exposure** – If `org.gradle.jvmargs` includes sensitive system properties, they may be logged. *Mitigation*: Avoid embedding secrets; use environment variables instead.

# Usage
```bash
# Verify which JVM args are applied
gradlew --status   # shows daemon info including JVM args

# Run a build with explicit GC settings (overrides gc.properties)
GRADLE_OPTS="-Xmx4g -XX:+UseG1GC" gradlew clean assemble

# Debug daemon startup (prints the full command line)
gradlew --no-daemon -Dorg.gradle.debug=true assemble
```

To edit the GC configuration:
1. Open `<project_root>/.gradle/6.7.1/gc.properties`.
2. Modify `org.gradle.jvmargs` (e.g., `-Xmx6g -XX:+UseG1GC -Xlog:gc*`).
3. Restart the daemon (`gradlew --stop`) or wait for idle timeout.

# Configuration
- **Environment Variables**
  - `JAVA_HOME` – Path to the JDK used by the wrapper.
  - `GRADLE_OPTS` – Additional JVM args that supersede those in `gc.properties`.
- **Config Files Referenced**
  - `gradle-wrapper.properties` (bootstrap version & distribution URL).
  - Project‑level `build.gradle` / `settings.gradle` (may set `org.gradle.jvmargs`).
  - Optional `gradle.properties` (global or user‑level) for default daemon args.

# Improvements
1. **Add explicit documentation header** – Include module‑specific recommendations (e.g., recommended heap size for Spark jobs) to prevent accidental mis‑configuration.
2. **Automate GC log rotation** – Append a `-Xlog:gc*:file=/var/log/gradle/gc-%p.log:time,uptime,level,tags:filecount=5,size