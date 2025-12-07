# Summary
Defines absolute filesystem locations for Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop‑CLI, generic JARs, and MNAAS‑specific JAR directories used by the production build of the MOVE‑Mediation Ingest (MNAAS) pipeline. The Ant build script reads these entries to construct compile‑time and runtime classpaths for production jobs.

# Key Components
- **hadoop.lib** – Path to core Hadoop libraries.  
- **hive.lib** – Path to Hive client libraries.  
- **hbase.lib** – Path to HBase client libraries.  
- **mapreduce.lib** – Path to Hadoop MapReduce libraries.  
- **hdfs.lib** – Path to Hadoop HDFS libraries.  
- **hadoopcli.lib** – Path to Hadoop CLI client libraries.  
- **generic.lib** – Path to generic third‑party JARs shared across pipelines.  
- **mnaas.generic.lib** – Path to MNAAS‑specific processing JARs (raw aggregation).  
- **mnaas.generic.load** – Path to MNAAS‑specific loader JARs.

# Data Flow
- **Input:** Ant build script (`build.xml`) reads this properties file at build time.  
- **Processing:** Values are concatenated to form classpath strings for `javac`, `jar`, and runtime execution of MapReduce/Hive jobs.  
- **Output:** Generated classpath environment variables; compiled JARs packaged with correct dependencies.  
- **Side Effects:** None beyond classpath construction.  
- **External Services/DBs/Queues:** Not directly accessed; downstream jobs may interact with HDFS, Hive, HBase, and message queues using the libraries referenced.

# Integrations
- **Ant (`build.xml`)** – Primary consumer; loads properties via `<property file=".../mnaas_prod.properties"/>`.  
- **Maven (`pom.xml`)** – May reference these paths for system‑scoped dependencies in production profiles.  
- **Runtime Jobs** – Hadoop, Hive, and HBase jobs launched by the pipeline inherit the classpath built from these entries.

# Operational Risks
- **Path Drift:** Hard‑coded absolute paths may become invalid after CDH upgrades or filesystem re‑layout. *Mitigation:* Use symlinks or environment‑driven base directory variables.  
- **Permission Issues:** Build agents must have read access to all directories. *Mitigation:* Enforce consistent ACLs and validate access in CI.  
- **Version Mismatch:** Libraries under these paths may be updated independently, causing incompatibilities. *Mitigation:* Pin versions via directory naming or checksum verification.

# Usage
```bash
# Example Ant invocation for production build
cd move-mediation-ingest/mnaas/bin
ant -f build.xml -Denv=prod clean compile jar
```
The Ant script will automatically load `mnaas_prod.properties` and construct the classpath.

# Configuration
- **File:** `move-mediation-ingest/mnaas/bin/META-INF/maven/com.tcl.move/mnaas/mnaas_prod.properties` (this file).  
- **Environment Variable (optional):** `MNAAS_CONF_DIR` can be set to override the default location of the properties file if the build script supports it.  
- **Related Configs:** `mnaas_dev.properties`, `mnaas.properties` for non‑production environments.

# Improvements
1. Replace absolute paths with a base directory variable (e.g., `${cdh.base}`) and compute sub‑paths to simplify upgrades.  
2. Externalize the properties into a version‑controlled YAML/JSON file and add a validation step in CI to detect missing or stale library versions.