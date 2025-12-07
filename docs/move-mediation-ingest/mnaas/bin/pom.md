# Summary
The `pom.xml` defines the Maven project for the MOVE‑Mediation Ingest (MNAAS) module. It declares build properties, repository access, and all compile‑time and runtime dependencies required to compile, package, and execute ingestion jobs within the telecom production pipeline.

# Key Components
- **Project Coordinates** – `groupId: com.tcl.move`, `artifactId: mnaas`, `version: 1.0`.
- **Properties** – Source encoding, Java 1.8 compiler target, source directory `${src.dir}`.
- **Repository** – Cloudera public repository for CDH artifacts.
- **Dependencies**  
  - `org.apache.hive:hive-jdbc:2.1.1-cdh6.3.3` – Hive JDBC driver.  
  - `com.tcl.move:hive_jdbc_connection:1.0` – Internal Hive connection wrapper.  
  - `com.sparkjava:spark-core:2.9.3` – Embedded web framework (REST API).  
  - `org.apache.spark:spark-sql_2.12:2.4.4` (provided) – Spark SQL API for data processing.  
  - `com.crealytics:spark-excel_2.12:0.14.0` – Excel file reader for Spark.  
  - `commons-logging:commons-logging:1.1.1` – Logging abstraction.  
  - `com.sun.mail:javax.mail:1.6.2` – Email notification support.  
  - `junit:junit:4.11` (test) – Unit‑test framework.  
  - `org.yaml:snakeyaml:1.28` – YAML parsing for configuration files.
- **Build Section** – Sets `${src.dir}` as the source directory.

# Data Flow
- **Inputs** – Java source files under `${src.dir}`; external property files (`mnaas.properties`, `mnaas_dev.properties`, `mnaas_prod.properties`).  
- **Outputs** – Compiled JAR (`mnaas-1.0.jar`), optionally a shaded/assembly JAR for deployment.  
- **Side Effects** – Downloads artifacts from Cloudera repo; populates local Maven repository; generates classpath used by Ant scripts and runtime containers.  
- **External Services** – Hive metastore (via JDBC), HDFS/HBase (via Spark), SMTP server (email), optional REST endpoints (SparkJava).

# Integrations
- **Ant Build (`build.xml`)** – Reads the generated JAR and classpath to construct compile‑time and runtime environments for ingestion jobs.  
- **Property Files** – `mnaas*.properties` supply absolute library paths; Maven dependencies provide the same libraries in a reproducible manner.  
- **Cloudera Repository** – Supplies CDH‑specific versions of Hive and Hadoop client libraries.  
- **Spark Cluster** – Runtime execution of Spark jobs packaged by this module.

# Operational Risks
- **Version Drift** – Fixed versions may become incompatible with cluster upgrades; mitigate by periodic dependency review.  
- **Provided Scope Misuse** – `spark-sql` is marked `provided`; runtime cluster must supply matching version, else `NoClassDefFoundError`.  
- **Repository Availability** – Cloudera repo outage blocks builds; add mirror or cache in internal Nexus/Artifactory.  
- **Transitive Conflicts** – Overlapping versions of logging or Hadoop libraries can cause classpath collisions; enforce dependency mediation with `<dependencyManagement>`.

# Usage
```bash
# Clean, compile, and package
mvn clean package

# Run unit tests
mvn test

# Install to local repository for Ant consumption
mvn install
```
Debug with Maven’s `-X` flag or attach a remote debugger to the Spark job after deployment.

# Configuration
- **Environment Variables** – None required by Maven; runtime may read `HADOOP_CONF_DIR`, `HIVE_CONF_DIR`.  
- **Config Files** – `mnaas.properties`, `mnaas_dev.properties`, `mnaas_prod.properties` (referenced by Ant scripts).  
- **Maven Settings** – May need proxy configuration for external repository access.

# Improvements
1. **Add Maven Enforcer Plugin** to enforce Java version, dependency convergence, and ban duplicate classes.  
2. **Introduce a Shade Plugin** to produce an uber‑JAR with all runtime dependencies, eliminating reliance on external property‑based classpaths.