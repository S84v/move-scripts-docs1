# Summary
`gradle-wrapper.properties` defines the Gradle Wrapper configuration for the **MoveGenevaInterface** module. It specifies the location and version (Gradle 6.7.1) of the Gradle distribution that will be automatically downloaded and used to build, test, and package the module in the telecom move‑mediation production pipeline.

# Key Components
- **distributionBase** – Base directory for storing wrapper files (`GRADLE_USER_HOME`).
- **distributionPath** – Relative path under `distributionBase` where the Gradle distribution zip is cached (`wrapper/dists`).
- **distributionUrl** – Fully qualified URL to the Gradle binary distribution (`gradle-6.7.1-bin.zip`).
- **zipStoreBase** – Base directory for the downloaded zip archive (`GRADLE_USER_HOME`).
- **zipStorePath** – Relative path under `zipStoreBase` where the zip is stored (`wrapper/dists`).

# Data Flow
- **Input:** No runtime input; the file is read by the Gradle Wrapper script (`gradlew`) during build initialization.
- **Output:** Downloads the specified Gradle distribution zip (if not present) and extracts it to the local wrapper cache.
- **Side Effects:** Populates `${GRADLE_USER_HOME}/wrapper/dists` with the Gradle distribution; may affect CI agents’ disk usage.
- **External Services:** HTTPS endpoint `https://services.gradle.org` for distribution download.

# Integrations
- **Gradle Wrapper (`gradlew` / `gradlew.bat`)** – Reads this file to resolve the Gradle version.
- **CI/CD pipelines (Jenkins, GitLab CI, etc.)** – Invoke `./gradlew`; the wrapper ensures a consistent Gradle version across build agents.
- **Project build scripts (`build.gradle`, `settings.gradle`)** – Executed by the resolved Gradle distribution.

# Operational Risks
- **Version Drift:** Using an outdated Gradle version may cause incompatibility with newer plugins or JDKs. *Mitigation:* Periodically review and upgrade the wrapper version.
- **Network Failure:** Failure to download the distribution blocks builds. *Mitigation:* Cache the zip in internal artifact repository or pre‑populate CI agents.
- **Security Exposure:** Unverified download URL could be tampered. *Mitigation:* Use checksum verification or host the distribution internally.

# Usage
```bash
# From the MoveGenevaInterface root directory
./gradlew clean build        # Wrapper downloads Gradle 6.7.1 if absent, then runs the build
./gradlew --version          # Verify the wrapper is using the expected Gradle version
```
For debugging wrapper download issues, inspect `${GRADLE_USER_HOME}/wrapper/dists` and the console logs.

# configuration
- **Environment Variable:** `GRADLE_USER_HOME` (defaults to `$HOME/.gradle` if unset).
- **Referenced Files:** None; all paths are relative to the wrapper configuration.

# Improvements
1. **Add SHA‑256 checksum** for the Gradle distribution to verify integrity on download.
2. **Upgrade to a supported Gradle LTS** (e.g., 7.x) and align plugin versions to reduce technical debt.