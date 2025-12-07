# Summary
The `compiler.xml` file defines the Java bytecode target level for each Gradle module within the **MoveGenevaInterface** project. In production builds, the IntelliJ IDEA compiler uses this configuration to ensure that compiled classes are compatible with Java 8, aligning with the runtime environment of the telecom move‑mediation system.

# Key Components
- **CompilerConfiguration component** – Holds compiler settings for the project.
- **bytecodeTargetLevel element** – Specifies the Java version (`1.8`) for each module.
  - `MoveGenevaInterface.main` – Main production source set.
  - `MoveGenevaInterface.test` – Unit‑test source set.
  - `MoveGenevaInterface_main` – Alternate naming for the main module (legacy).
  - `MoveGenevaInterface_test` – Alternate naming for the test module (legacy).

# Data Flow
- **Inputs:** IntelliJ IDEA project model, Gradle module definitions, Java Development Kit (JDK) 8 installation.
- **Outputs:** Compiled `.class` files targeting JVM bytecode version 52.0 (Java 8) placed in `build/classes/java/main` and `build/classes/java/test`.
- **Side Effects:** Enforces compatibility checks during incremental compilation; may trigger recompilation if target level mismatches.
- **External Services/DBs/Queues:** None.

# Integrations
- **Gradle Wrapper (`gradlew`)** – Reads module structure; the IDE compiler settings must match Gradle’s `sourceCompatibility`/`targetCompatibility` to avoid build divergence.
- **CI/CD pipelines** – Use the same JDK 8 version; the file ensures IDE‑local builds mirror pipeline builds.
- **Telecom Move System scripts** – Deploy compiled artifacts (`*.jar`) produced from these modules to the MoveGenevaInterface runtime environment.

# Operational Risks
- **Version drift:** If Gradle `targetCompatibility` diverges from `compiler.xml`, builds may succeed locally but fail in CI. *Mitigation:* Enforce consistency via a Gradle check task.
- **JDK mismatch:** Using a JDK newer than 8 can produce bytecode incompatible with target level. *Mitigation:* Pin JDK version in CI and developer environments.
- **Legacy module names:** Duplicate entries (`*_main`, `*_test`) may cause confusion. *Mitigation:* Consolidate to a single canonical module name.

# Usage
1. Open the project in IntelliJ IDEA.
2. Ensure JDK 8 is selected for the project SDK.
3. Trigger **Build → Rebuild Project**; the IDE reads `compiler.xml` to set the bytecode target.
4. Verify compiled classes are version 52.0 using `javap -verbose <class>`.

# configuration
- **Environment Variables:** `JAVA_HOME` must point to JDK 8.
- **Referenced Config Files:** `build.gradle` (or `build.gradle.kts`) where `sourceCompatibility = JavaVersion.VERSION_1_8` and `targetCompatibility = JavaVersion.VERSION_1_8` are declared.

# Improvements
- **TODO 1:** Add a Gradle verification task (`checkCompilerConfig`) that parses `compiler.xml` and fails the build if target levels differ from Gradle settings.
- **TODO 2:** Remove redundant legacy module entries (`*_main`, `*_test`) to simplify the configuration and reduce maintenance overhead.