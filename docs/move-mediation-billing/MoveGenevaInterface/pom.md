# Summary
The `pom.xml` defines the Maven project for the **MoveGenevaInterface** module of the telecom move‑system. It configures build properties, declares required third‑party libraries (Hive JDBC, Spark, Spark‑Excel, Mail, YAML, etc.), and points to the Cloudera artifact repository. In production the module is compiled with Java 17, packaged, and deployed as part of the Move‑Geneva integration layer that ingests billing data, processes it with Spark, and forwards results to downstream services.

# Key Components
- **Project coordinates**: `groupId=com.tcl.move`, `artifactId=mnaas`, `version=1.0`.
- **Properties**
  - `project.build.sourceEncoding` – UTF‑8.
  - `maven.compiler.source` / `maven.compiler.target` – Java 17.0.6.
  - `src.dir` – source root (`src`).
- **Repositories**
  - Cloudera public repository (`https://repository.cloudera.com/artifactory/cloudera-repos/`).
- **Dependencies**
  - `org.apache.hive:hive-jdbc:3.1.3000.7.2.15.0-147` (excludes `jdk.tools`).
  - `com.sparkjava:spark-core:2.9.3` (embedded HTTP server).
  - `org.apache.spark:spark-sql_2.12:2.4.4` (scope **provided** – supplied by runtime cluster).
  - `com.crealytics:spark-excel_2.12:0.14.0` (Excel I/O for Spark).
  - `commons-logging:commons-logging:1.1.1`.
  - `com.sun.mail:javax.mail:1.6.2` (e‑mail alerts).
  - `junit:junit:4.11` (test scope).
  - `org.yaml:snakeyaml:1.28` (YAML configuration parsing).
- **Commented system‑scoped dependency** (Hive JDBC connection JAR) – retained for reference only.

# Data Flow
| Phase | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| **Build** | `pom.xml`, source files under `${src.dir}` | Maven compiler plugin (Java 17) resolves dependencies from Maven Central and Cloudera repo, compiles code, runs unit tests (JUnit) | Compiled classes, JAR/assembly artifact |
| **Runtime** | External data (e.g., Hive tables, Excel files, HTTP requests) | Spark SQL jobs (provided by cluster) read/write Hive via `hive-jdbc`; Spark‑Excel reads/writes Excel; Spark‑Core may expose REST endpoints; `javax.mail` sends alerts; `snakeyaml` parses config files | Processed billing records, persisted results, alert e‑mails |
| **Testing** | Test sources | JUnit executes unit tests | Test reports, code coverage metrics |

External services accessed at runtime:
- **Hive Metastore / JDBC** (via `hive-jdbc`).
- **Spark cluster** (provided runtime, Spark‑SQL, Spark‑Excel).
- **SMTP server** (via `javax.mail`).
- **File system** for Excel/YAML resources.

# Integrations
- **Balance Notification components** (`NotificationService`, `JSONParseUtils`) depend on `javax.mail` for alerting and may use `snakeyaml` for configuration; they are built with this POM.
- **Kafka consumer** (outside this module) produces JSON payloads that are parsed and persisted using the same libraries.
- **Gradle wrapper** (`gradlew.bat`) in the sibling `MoveGenevaInterface` directory can invoke Maven goals via the `exec` plugin (not shown) or as part of a larger CI pipeline.
- **Cloudera repository** supplies the Hive JDBC driver required for data ingestion from Hive tables.

# Operational Risks
- **Version incompatibility**: Spark‑SQL 2.4.4 may conflict with newer Hive JDBC driver or Java 17; verify compatibility in the target Spark cluster.
- **Provided‑scope dependency**: `spark-sql` must be present on every execution node; missing or mismatched versions cause `NoClassDefFoundError`.
- **External repository availability**: Cloudera repo outage blocks builds; mitigate with a local mirror or cache.
- **Exclusion of `jdk.tools`** may cause compilation failures if tools are required at runtime; monitor build logs.
- **Unpinned transitive dependencies** (e.g., commons‑logging) could introduce security CVEs; enforce dependency convergence.

# Usage
```bash
# Clean, compile, and package the module
mvn clean compile package

# Run unit tests
mvn test

# Skip tests (e.g., during CI)
mvn -DskipTests clean package

# Verify dependency tree for conflicts
mvn dependency:tree -Dverbose
```
To debug runtime issues, attach a remote debugger to the Spark driver or use `spark-submit --master local[*]` with the built JAR.

# Configuration
- **Environment variables**
  - `JAVA_HOME` – must point to a JDK 17 installation.
  - `MAVEN_OPTS` – optional JVM options for Maven (e.g., memory settings).
- **External config files**
  - YAML files parsed by `snakeyaml` (path supplied by application code).
  - SMTP host/port credentials (usually via environment variables or a properties file read at runtime).
- **Repository credentials** (if required) are managed in `~/.m2/settings.xml`.

# Improvements
1. **Upgrade dependencies**: Move to Spark 3.x (compatible with Hive 3) and update `hive-jdbc` to a newer, Java 17‑compatible version. Align all libraries to the same Scala version.
2. **Add a `<dependencyManagement>` section** to lock transitive versions and enforce a corporate BOM, reducing risk of version drift and simplifying future upgrades.