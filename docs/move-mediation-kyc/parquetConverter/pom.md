# Summary
The `pom.xml` defines the Maven project for the **parquetConverter** utility within the MOVE‑KYC subsystem. It packages the source as a JAR that provides Apache Parquet (Avro and Hadoop) support for reading/writing Parquet files in the telecom data‑processing pipeline.

# Key Components
- **Project coordinates**: `groupId=com.tcl.parquet`, `artifactId=utility`, `version=0.0.1‑SNAPSHOT`.
- **Packaging**: `jar`.
- **Build properties**: source encoding set to UTF‑8.
- **Dependencies**:
  - `junit:junit:3.8.1` (test scope).
  - `org.apache.parquet:parquet-avro:1.8.1`.
  - `org.apache.parquet:parquet-hadoop:1.8.1`.
  - `org.apache.hadoop:hadoop-common:3.0.0`.
- **Maven metadata**: project name, URL, model version.

# Data Flow
| Phase | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| **Compile** | Java source (`src/main/java`) | Maven compiler (default) resolves dependencies, compiles classes against Parquet/Hadoop APIs | `target/classes` | Downloads artifacts from remote repositories |
| **Package** | Compiled classes, resources | Maven JAR plugin creates `utility-0.0.1-SNAPSHOT.jar` | `target/utility-0.0.1-SNAPSHOT.jar` | None |
| **Test** (optional) | Test sources (`src/test/java`) | JUnit 3 runner executes unit tests | Test reports (`target/surefire-reports`) | None |

External services: Hadoop client libraries (runtime) and Parquet libraries; no direct DB or queue interaction in this module.

# Integrations
- Consumed by other MOVE modules (e.g., `move-mediation-kyc`) that need to convert Hadoop‑based datasets to/from Parquet.
- Runtime classpath must include Hadoop configuration files (`core-site.xml`, `hdfs-site.xml`) for HDFS access.
- May be invoked from batch scripts or Spark jobs that reference the generated JAR.

# Operational Risks
- **Dependency version drift**: Parquet 1.8.1 may be incompatible with newer Hadoop clusters. *Mitigation*: Align Parquet version with Hadoop distribution; add dependencyManagement.
- **JUnit 3** is obsolete; tests may not run on modern JDKs. *Mitigation*: Upgrade to JUnit 4/5.
- **Missing Hadoop configuration** at runtime leads to `FileSystem` initialization failures. *Mitigation*: Ensure Hadoop config files are on the classpath or supplied via `-D` properties.
- **Snapshot version** may be unintentionally promoted to production. *Mitigation*: Use release versions and enforce CI gating.

# Usage
```bash
# Build the JAR
mvn clean package

# Run unit tests (if any)
mvn test

# Install to local repository for other modules
mvn install
```
To debug, attach a remote debugger to the JVM executing the utility or run `mvn -X` for Maven debug output.

# Configuration
- **Maven settings** (`~/.m2/settings.xml`) for repository credentials if required.
- **Runtime Hadoop config**: `HADOOP_CONF_DIR` or explicit `-D` system properties (`fs.defaultFS`, `dfs.replication`, etc.).
- No environment variables referenced directly in the POM.

# Improvements
1. **Upgrade dependencies**: Move to Parquet 1.12.x and Hadoop 3.3.x; update JUnit to 5.x; add explicit version constraints via `<dependencyManagement>`.
2. **Add Maven Shade plugin** to produce an uber‑JAR that bundles Hadoop and Parquet dependencies, simplifying deployment on nodes without a full Hadoop client installation.