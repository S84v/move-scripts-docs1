# Summary
`gradlew.bat` is the Windows bootstrap script for the Gradle Wrapper used by the **MoveGenevaInterface** module. It resolves the Java runtime, applies default and user‑provided JVM options, constructs the classpath to `gradle-wrapper.jar`, and launches the Gradle daemon to execute build tasks (e.g., compile, test, package) required for the telecom move‑system production artifacts.

# Key Components
- **Environment detection**
  - Checks `JAVA_HOME`; falls back to `java` on `PATH`.
- **Default JVM options**
  - `-Xmx64m -Xms64m` defined in `DEFAULT_JVM_OPTS`.
- **User‑override variables**
  - `JAVA_OPTS`, `GRADLE_OPTS` appended to the JVM command line.
- **Classpath construction**
  - Sets `CLASSPATH` to `<APP_HOME>\gradle\wrapper\gradle-wrapper.jar`.
- **Execution block**
  - Invokes `java.exe` with assembled options and the `GradleWrapperMain` class.
- **Error handling**
  - Validates Java availability; prints descriptive errors; exits with non‑zero status.
- **Shutdown handling**
  - Returns appropriate exit code to the calling shell.

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| 1 | Environment variables (`JAVA_HOME`, `JAVA_OPTS`, `GRADLE_OPTS`, `DEBUG`) | Resolve Java executable; assemble JVM options | Selected `java.exe` path |
| 2 | Script arguments (`%*`) | Forward to Gradle Wrapper | Gradle task execution (e.g., `clean`, `build`) |
| 3 | `gradle-wrapper.jar` | Loaded by JVM as classpath | Invocation of `org.gradle.wrapper.GradleWrapperMain` |
| 4 | Gradle build scripts (`build.gradle`, settings) | Executed by Gradle | Compiled classes, JAR/WAR artifacts, test reports |
| 5 | Console / log | Printed messages for success or failure | Exit code propagated to CI/CD pipeline |

External services: none directly; relies on local JDK and file system.

# Integrations
- **Gradle Wrapper JAR** (`gradle-wrapper.jar`) – bundled with the repository; ensures consistent Gradle version.
- **Java Runtime** – required for all build steps; interacts with system `JAVA_HOME`.
- **CI/CD pipelines** – invoked by build agents to compile and package the MoveGenevaInterface component.
- **Source control** – script resides in the repository; executed after checkout.

# Operational Risks
- **Missing or mis‑configured `JAVA_HOME`** → build failure; mitigate by enforcing JDK installation checks in CI agents.
- **Insufficient JVM heap (`-Xmx64m`) for large builds** → OutOfMemoryError; mitigate by overriding `JAVA_OPTS` or `GRADLE_OPTS` with higher memory settings.
- **Path length limitations** on Windows may truncate `APP_HOME`; mitigate by placing repository on a short path (e.g., `C:\repo`).
- **Wrapper version drift** – outdated `gradle-wrapper.jar` may cause incompatibility; mitigate by periodic wrapper upgrade and checksum verification.

# Usage
```bat
rem Navigate to the MoveGenevaInterface root
cd move-mediation-billing\MoveGenevaInterface

rem Clean and build the module
gradlew.bat clean build

rem Run with custom JVM memory (e.g., 512 MB)
set JAVA_OPTS=-Xmx512m -Xms256m
gradlew.bat assemble
```
For debugging, set `DEBUG=1` before execution to retain console output.

# configuration
- **Environment Variables**
  - `JAVA_HOME` – absolute path to JDK installation (required if `java` not on `PATH`).
  - `JAVA_OPTS` – additional JVM arguments (optional).
  - `GRADLE_OPTS` – additional arguments passed to Gradle (optional).
  - `DEBUG` – when set, script echoes commands.
- **Internal Constants**
  - `DEFAULT_JVM_OPTS` – `"-Xmx64m" "-Xms64m"` (can be overridden via `JAVA_OPTS`).

# Improvements
1. **Dynamic memory sizing** – detect available system memory and set `-Xmx` accordingly, falling back to user‑provided overrides.
2. **Enhanced logging** – redirect Gradle console output to a timestamped log file for post‑mortem analysis, and expose a `--log-file` option.