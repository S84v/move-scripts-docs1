# Summary
`gc.properties` located in `move-mediation-billing\MoveGenevaInterface\.gradle\vcs-1\` is a Gradle‑generated configuration file that records the JVM garbage‑collection (GC) arguments used by the Gradle daemon when the **vcs‑1** (Version‑Control‑System‑1) cache is active. In production the Gradle Wrapper reads this file each time a daemon is started for the *MoveGenevaInterface* module, ensuring that the same GC tuning (heap size, GC algorithm, logging) is applied across builds, tests, and packaging steps, thereby providing deterministic memory‑management behavior and preventing daemon‑restart‑induced build variability.

# Key Components
- **Gradle Daemon** – launches with JVM args read from `gc.properties`.
- **Gradle Wrapper (`gradlew` / `gradlew.bat`)** – resolves the file path and injects the properties into the daemon command line.
- **vcs‑1 cache subsystem** – associates the GC settings with the version‑control‑system‑1 cache directory to keep daemon configuration scoped per cache variant.

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| Daemon start | `gradlew` invocation, environment variables, `gradle.properties` | Wrapper locates `...\.gradle\vcs-1\gc.properties` and parses key‑value pairs (e.g., `org.gradle.jvmargs=-Xmx2g -XX:+UseG1GC`) | JVM launched with specified GC args; daemon registers the settings for subsequent builds |
| Cache validation | Cached daemon metadata | Gradle compares current GC args with those stored in `gc.properties` | If mismatch → daemon shutdown & restart with new args |
| Build execution | Source code, dependencies | Uses the daemon with the configured GC settings | Build artifacts, logs, potential GC logs (if enabled) |

External services: none; interacts only with local file system and Gradle daemon.

# Integrations
- **Other `.gradle` gc.properties files** (`.gradle/6.7.1/gc.properties`, `.gradle/configuration-cache/gc.properties`) – provide version‑wide or feature‑specific GC settings; the `vcs-1` file overrides when the vcs‑1 cache is selected.
- **CI/CD pipelines** – pipeline scripts invoke `./gradlew`; the presence of this file ensures consistent daemon behavior across agents.
- **Monitoring tools** – if GC logging is enabled, log files are written to the daemon’s working directory and consumed by log aggregation services.

# Operational Risks
- **Stale GC configuration** – outdated heap size may cause OOM during large builds. *Mitigation*: automate regeneration of `gc.properties` on Gradle version upgrade or when JVM args change.
- **Daemon mismatch** – if the file is missing or corrupted, Gradle falls back to default JVM args, potentially degrading performance. *Mitigation*: add health‑check step in build scripts to verify file existence and parseability.
- **Security exposure** – GC args may include debug flags that leak internal state. *Mitigation*: restrict file permissions to the build service account.

# Usage
```bash
# Verify current GC settings
cat move-mediation-billing/MoveGenevaInterface/.gradle/vcs-1/gc.properties

# Force regeneration (e.g., after changing JVM args in gradle.properties)
./gradlew --stop
rm -f move-mediation-billing/MoveGenevaInterface/.gradle/vcs-1/gc.properties
./gradlew clean build   # daemon will recreate the file with current args
```

# configuration
- **Environment Variables**
  - `GRADLE_USER_HOME` – may redirect the `.gradle` directory; ensure it points to the module’s `.gradle` folder.
  - `ORG_GRADLE_PROJECT_org.gradle.jvmargs` – can be set to override GC args; values are persisted to `gc.properties`.
- **Referenced Config Files**
  - `gradle.properties` (project‑level) – defines `org.gradle.jvmargs`.
  - `settings.gradle` – may enable the `vcs-1` cache feature.

# Improvements
1. **Automated validation** – add a Gradle task (`validateGcProperties`) that parses `gc.properties` and fails the build if required keys are missing or values exceed safe thresholds.
2. **Version stamping** – embed the Gradle version and timestamp in the file (e.g., `gradleVersion=6.7.1`, `generatedAt=2025-12-05T12:00:00Z`) to aid troubleshooting and cache compatibility checks.