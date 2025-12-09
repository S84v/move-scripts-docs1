# Summary
`pom.xml` defines the Maven project for the **tclcustomreport** component of the Move‑Mediation revenue‑assurance system. It configures build parameters, resolves external libraries (Hive JDBC, Spark‑Excel, mail, YAML, logging), and produces a deployable JAR that is consumed by batch jobs generating custom telecom reports.

# Key Components
- **Project coordinates** – `groupId: tclcustomreport`, `artifactId: tclcustomreport`, `version: 0.0.1‑SNAPSHOT`.
- **Build properties** – source/target Java 1.8, UTF‑8 encoding, `src.dir` path.
- **Repository** – Cloudera public repository for CDH‑specific artifacts.
- **Dependencies**
  - `org.apache.hive:hive-jdbc:2.1.1-cdh6.3.3` – Hive connectivity.
  - `com.tcl.move:hive_jdbc_connection:1.0` – Internal wrapper for Hive connections.
  - `com.crealytics:spark-excel_2.12:0.14.0` – Spark‑Excel writer/reader.
  - `commons-logging:commons-logging:1.1.1` – Logging façade.
  - `com.sun.mail:javax.mail:1.6.2` – Email notification support.
  - `org.yaml:snakeyaml:1.28` – YAML configuration parsing.
  - `junit:junit:4.11` & `junit:junit:3.8.1` – Test frameworks (both scopes `test`).
- **Packaging** – JAR.

# Data Flow
| Phase | Input | Process | Output | Side Effects / External Interaction |
|-------|-------|---------|--------|------------------------------------|
| **Compile** | Java source under `${src.dir}` | Maven Compiler Plugin (implicit) using Java 1.8 | `.class` files in `target/classes` | None |
| **Package** | Compiled classes + resources | Maven JAR plugin (implicit) | `tclcustomreport-0.0.1-SNAPSHOT.jar` | None |
| **Runtime** (when JAR is used) | Configuration files (YAML, properties), Hive tables, Spark jobs, SMTP server | Service classes (e.g., `MediaitionFetchService`, `SCPService`, `Utils`) use the libraries declared here | Report files (CSV/Excel), email notifications, remote SCP transfer | Hive DB, Hadoop cluster, SMTP server, remote SSH/SCP endpoint |

# Integrations
- **Hive/JDBC** – `hive-jdbc` and internal `hive_jdbc_connection` enable `MediaitionFetchDAO` to execute Hive queries.
- **Spark‑Excel** – Used by report generation utilities to write Excel files.
- **Mail Service** – `javax.mail` supports `Utils`/`MailService` for alerting on failures.
- **YAML** – `snakeyaml` parses external configuration (e.g., `application.yml`) consumed by services.
- **Logging** – `commons-logging` bridges to the underlying logging framework (Log4j/SLF4J) used across the codebase.
- **Test Suites** – JUnit dependencies allow unit tests for service classes.

# Operational Risks
- **Dependency version drift** – Mixed JUnit versions (4.11 & 3.8.1) may cause classpath conflicts. *Mitigation*: Consolidate to a single, up‑to‑date JUnit version.
- **Outdated libraries** – Hive JDBC 2.1.1 and Commons‑Logging 1.1.1 are several years old; security patches may be missing. *Mitigation*: Evaluate newer CDH/Hive client versions and upgrade logging façade.
- **Transitive conflicts** – `spark-excel_2.12` pulls Scala 2.12 artifacts; ensure the runtime Spark cluster matches this Scala version. *Mitigation*: Align Spark/Scala versions across the environment.
- **Hard‑coded repository** – Reliance on the public Cloudera repo may cause build failures if the repo is unavailable. *Mitigation*: Mirror required artifacts in an internal Nexus/Artifactory.

# Usage
```bash
# Build the JAR
mvn clean package

# Run unit tests (if any)
mvn test

# Execute a specific class (example)
java -cp target/tclcustomreport-0.0.1-SNAPSHOT.jar com.tcl.move.service.MediaitionFetchService
```
For debugging inside an IDE, import the Maven project; ensure the `src` directory is set to `${project.basedir}/src`.

# Configuration
- **Environment variables** – Not defined in this POM; runtime services read:
  - Hive connection parameters (host, port, auth) from external YAML/Properties.
  - SMTP host/credentials for email alerts.
  - SSH/SCP credentials for `SCPService`.
- **Config files** – Typically `application.yml` or `config.properties` located on the classpath; parsed via SnakeYAML or Java `Properties`.

# Improvements
1. **Dependency hygiene** – Remove duplicate JUnit entries, upgrade to JUnit 5, and align all test libraries to a single version.
2. **Version management** – Introduce a `<dependencyManagement>` section or a Maven BOM to centralize versions (Hive, Spark‑Excel, logging) and simplify future upgrades.