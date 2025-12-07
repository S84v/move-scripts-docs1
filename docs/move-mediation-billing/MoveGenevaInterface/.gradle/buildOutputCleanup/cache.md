# Summary
`cache.properties` is a Gradle‑generated metadata file located in `<module>\.gradle\buildOutputCleanup`. It records the Gradle version (`6.7.1`) used for the last successful build and the timestamp of its creation. In production the Gradle Build‑Output‑Cleanup service reads this file to decide whether cached build outputs are compatible with the current Gradle runtime and to purge stale artifacts, ensuring reproducible builds of the **MoveGenevaInterface** module.

# Key Components
- **Gradle Build‑Output‑Cleanup** – internal Gradle service that scans `<module>\.gradle\buildOutputCleanup` directories.
- `cache.properties` – plain‑text key/value store:
  - `gradle.version` – version of Gradle that produced the cached outputs.
  - Timestamp comment – creation time (used for age‑based eviction).
- **Gradle Daemon** – reads the file on startup to validate cache compatibility.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | Existing `cache.properties` (timestamp, version) | Gradle daemon compares `gradle.version` with the wrapper‑specified version (`6.7.1`). | If mismatch → cache invalidated; otherwise cache retained. |
| 2 | Current system time | Used to compute age of cached outputs. | Age‑based cleanup (deletes outputs older than configured threshold). |
| 3 | Cleaned cache directories | Gradle rebuilds missing artifacts. | New `cache.properties` written with updated timestamp. |

No external services, databases, or message queues are involved.

# Integrations
- **gradlew.bat / gradlew** – invokes the Gradle Wrapper, which sets `GRADLE_USER_HOME` to `<module>\.gradle` and triggers the Build‑Output‑Cleanup service.
- **.gradle/6.7.1/** – version‑specific cache directory; `cache.properties` links the cleanup logic to this version.
- **MoveGenevaInterface build scripts (pom.xml, build.gradle)** – indirectly depend on the cleanup to guarantee that compiled Spark/Hive artifacts are built against the correct Gradle runtime.

# Operational Risks
- **Version Drift**: If the wrapper is upgraded without cleaning the `.gradle` directory, stale caches may cause build failures. *Mitigation*: enforce `gradlew cleanBuildCache` after wrapper version changes.
- **Cache Corruption**: Manual edits or disk errors corrupt `cache.properties`. *Mitigation*: monitor file integrity; recreate cache on detection.
- **Excessive Retention**: Overly aggressive retention may fill disk space on build agents. *Mitigation*: configure `org.gradle.caching=true` and set `org.gradle.caching.cleanup` policies.

# Usage
```cmd
rem Validate current cache version
gradlew --status

rem Force cleanup (recreates cache.properties)
gradlew cleanBuildCache
```
The commands read/write `cache.properties` automatically; no direct manipulation is required.

# configuration
- **Environment Variables**
  - `GRADLE_USER_HOME` – defaults to `<module>\.gradle`; can be overridden to relocate caches.
- **Gradle Properties**
  - `org.gradle.caching=true` – enables build‑output caching.
  - `org.gradle.caching.cleanup` – defines age/size thresholds for automatic cleanup (e.g., `org.gradle.caching.cleanup=30d`).

# Improvements
1. **Automated Version Guard** – add a pre‑build check that aborts if `cache.properties` version differs from the wrapper version, with a clear error message.
2. **Retention Policy Configuration** – expose a module‑level property (e.g., `moveGeneva.cache.maxAgeDays`) to allow ops to tune cleanup without modifying global Gradle settings.