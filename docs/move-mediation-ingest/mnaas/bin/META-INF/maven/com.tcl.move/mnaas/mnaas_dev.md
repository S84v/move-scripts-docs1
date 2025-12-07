# Summary
`mnaas_dev.properties` supplies absolute filesystem locations for Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop‑CLI, generic, and MNAAS‑specific JAR directories. The Ant build script (`build.xml`) reads these entries to assemble compile‑time and runtime classpaths for the MOVE‑Mediation Ingest (MNAAS) development pipeline.

# Key Components
- **hadoop.lib** – Path to Hadoop core libraries.  
- **hive.lib** – Path to Hive client libraries.  
- **hbase.lib** – Path to HBase client libraries.  
- **mapreduce.lib** – Path to Hadoop MapReduce libraries.  
- **hdfs.lib** – Path to Hadoop HDFS libraries.  
- **hadoopcli.lib** – Path to Hadoop CLI libraries.  
- **generic.lib** – Path to shared generic JARs used across ingestion jobs.  
- **mnaas.generic.lib** – Path to MNAAS‑specific processing JARs (raw aggregation).  
- **mnaas.generic.load** – Path to MNAAS loader JARs.

# Data Flow
- **Input:** Ant build script (`build.xml`) reads the properties file.  
- **Processing:** Values are concatenated into classpath strings for `javac`, `jar`, and runtime execution of ingestion jobs.  
- **Output:** Generated classpath environment variables; compiled JARs and job submission scripts.  
- **Side Effects:** None beyond classpath construction.  
- **External Services/DBs/Queues:** Not directly accessed; downstream jobs may interact with HDFS, Hive, HBase, and message queues.

# Integrations
- **Ant (`build.xml`)** – Primary consumer; uses `${property.file}` to load entries.  
- **Maven (`pom.xml`)** – May reference the same directories for manual overrides.  
- **Runtime ingestion jobs** – Rely on the classpaths built from these paths to locate Hadoop/Hive/HBase APIs.  
- **Deployment scripts** – May copy or symlink this file to target nodes.

# Operational Risks
- **Hard‑coded absolute paths** – Break if CDH installation moves or version changes. *Mitigation:* Parameterize base directory via environment variable.  
- **Environment drift** – Different nodes may have divergent directory structures. *Mitigation:* Validate paths at build start; fail fast with clear error.  
- **Permission issues** – Build may lack read access to the directories. *Mitigation:* Ensure build user belongs to appropriate groups.  
- **Stale JAR versions** – Paths may point to outdated libraries. *Mitigation:* Periodic audit against CDH version.

# Usage
```bash
# From the mnaas/bin directory
ant -f build.xml clean compile jar
# Ant will load mnaas_dev.properties automatically (property file location defined in build.xml)
```
To debug classpath composition:
```bash
ant -f build.xml -Ddebug.classpath=true
```

# Configuration
- **File:** `move-mediation-ingest/mnaas/bin/META-INF/maven/com.tcl.move/mnaas/mnaas_dev.properties` (current file).  
- **Referenced by:** `build.xml` property task (`<property file=".../mnaas_dev.properties"/>`).  
- **Environment variables (optional):** `CDH_BASE=/app/cloudera/opt/cloudera/parcels/CDH` can be used to replace repeated prefix.  

# Improvements
1. **Externalize base directory:** Replace repeated `/app/cloudera/opt/cloudera/parcels/CDH` with `${cdh.base}` resolved from an environment variable or a separate properties file.  
2. **Add validation target:** Introduce an Ant target that checks existence and readability of each path before compilation, aborting with a descriptive message if any path is invalid.