# Summary
The `pom.xml` defines the Maven project for the MOVE‑Mediation Ingest (MNAAS) module. It declares build settings, repository sources, and all compile‑time, provided, and test dependencies required to compile, package, and run the ingestion pipeline in the telecom production environment.

# Key Components
- **Project coordinates** – `groupId: com.tcl.move`, `artifactId: mnaas`, `version: 1.0`.
- **Properties** – Source encoding, Java 1.8 compiler target/source, source directory (`src`).
- **Repository** – Cloudera public repository for CDH‑specific artifacts.
- **Dependencies**
  - `org.apache.hive:hive-jdbc:2.1.1-cdh6.3.3` – Hive JDBC driver.
  - `com.tcl.move:hive_jdbc_connection:1.0` – Internal wrapper for Hive connections.
  - `com.sparkjava:spark-core:2.9.3` – Embedded web framework (REST API).
  - `org.apache.spark:spark-sql_2.12:2.4.4` (provided) – Spark SQL runtime.
  - `com.crealytics:spark-excel_2.12:0.14.0` – Excel file reader for Spark.
  - `commons-logging:commons-logging:1.1.1` – Logging façade.
  - `com.sun.mail:javax.mail:1.6.2` – Email notification support.
  - `org.yaml:snakeyaml:1.28` – YAML configuration parsing.
  - `jdk.tools:jdk.tools:1.8.0_221` (system) – Access to `tools.jar` for compiler APIs.
  - `junit:junit:4.11` (test) – Unit‑test framework.
- **Build configuration** – Overrides default source directory to `${src.dir}`.

# Data Flow
- **Inputs**: Source code under `src/`; external JARs resolved from Maven Central, Cloudera repo, and local `tools.jar`.
- **Outputs**: Compiled classes, JAR artifact `mnaas-1.0.jar`; optionally a shaded/uber‑jar if downstream packaging is applied.
- **Side Effects**: Downloads of remote artifacts; local cache population in `~/.m2/repository`.
- **External Services**: Cloudera repository (HTTPS); local JDK installation for `tools.jar`.

# Integrations
- **Ant build (`build.xml`)** – The Ant scripts referenced in the surrounding documentation read the same property files (`mnaas*.properties`) to construct classpaths; the Maven build produces the same JAR that Ant may later package.
- **Hive/Hadoop clusters** – Runtime classpath includes Hive JDBC and Spark SQL, enabling connections to Hive Metastore and execution on Hadoop YARN.
- **Email subsystem** – `javax.mail` used to send status or error notifications.
- **REST API** – `spark-core` provides HTTP endpoints consumed by monitoring or orchestration tools.
- **Excel ingestion** – `spark-excel` reads Excel workbooks supplied by upstream data feeds.

# Operational Risks
- **Version drift** – Fixed versions (e.g., Hive 2.1.1‑cdh6.3.3) may become incompatible with upgraded CDH clusters; mitigate by aligning Maven versions with cluster libraries.
- **System‑scoped `tools.jar`** – Hard‑coded Windows path (`C:/Program Files/Java/...`) breaks on non‑Windows agents; mitigate by using the Maven `maven-compiler-plugin` instead of system dependency.
- **Provided scope for Spark** – Production runtime must supply matching Spark libraries; missing or mismatched Spark version causes `NoClassDefFoundError`. Ensure cluster Spark version matches `2.4.4`.
- **Transitive dependency conflicts** – Multiple logging frameworks (commons‑logging vs SLF4J) may cause duplicate bindings; enforce a single logging implementation via dependencyManagement.

# Usage
```bash
# Compile and package
mvn clean package

# Run unit tests
mvn test

# Execute the JAR (requires Spark and Hadoop libraries on classpath)
java -cp target/mnaas-1.0.jar:<spark-lib-dir>/* com.tcl.move.mnaas.Main
```
Replace `<spark-lib-dir>` with the directory containing Spark runtime JARs on the target node.

# Configuration
- **Property files**: `mnaas.properties`, `mnaas_dev.properties`, `mnaas_prod.properties` – external classpath definitions used by Ant; not directly read by Maven but required for deployment.
- **Environment variables** (implicit):
  - `JAVA_HOME` – points to JDK 1.8 (required for `tools.jar`).
  - `M2_HOME` / `MAVEN_OPTS` – standard Maven settings.
- **System properties** (if needed at runtime):
  - `hadoop.conf.dir`, `hive.conf.dir` – paths to Hadoop/Hive configuration files.

# Improvements
1. **Replace system‑scoped `jdk.tools` dependency** with the `maven-compiler-plugin` configuration to avoid OS‑specific paths and improve portability.
2. **Introduce a dependencyManagement section** to lock transitive versions (e.g., logging, Jackson) and add explicit exclusions for known conflicts, ensuring reproducible builds across environments.