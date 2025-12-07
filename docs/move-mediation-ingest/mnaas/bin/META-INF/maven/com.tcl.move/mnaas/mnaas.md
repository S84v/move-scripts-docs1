# Summary
`mnaas.properties` supplies absolute filesystem locations for Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop‑CLI, generic, and MNAAS‑specific JAR directories. The Ant build script (`build.xml`) reads these entries to construct compile‑time and runtime classpaths for the MOVE‑Mediation Ingest (MNAAS) production pipeline.

# Key Components
- **hadoop.lib** – Path to Hadoop core libraries.  
- **hive.lib** – Path to Hive client libraries.  
- **hbase.lib** – Path to HBase client libraries.  
- **mapreduce.lib** – Path to Hadoop MapReduce libraries.  
- **hdfs.lib** – Path to Hadoop HDFS libraries.  
- **hadoopcli.lib** – Path to Hadoop CLI client libraries.  
- **generic.lib** – Path to shared generic JARs used across ingestion jobs.  
- **mnaas.generic.lib** – Path to MNAAS‑specific processing JARs (e.g., raw aggregation tables).  
- **mnaas.generic.load** – Path to MNAAS loader JARs.

# Data Flow
- **Input:** Ant build script (`build.xml`) reads the properties file.  
- **Processing:** Values are concatenated into classpath strings for `javac`, `jar`, and runtime `java` commands.  
- **Output:** Generated classpath environment variables (`classpath`, `runtime.classpath`) passed to compilation, packaging, and job submission steps.  
- **Side Effects:** None; file is read‑only at build time.  
- **External Services/DBs/Queues:** Not directly accessed; classpath entries may reference JARs that interact with HDFS, Hive, HBase, or message queues at runtime.

# Integrations
- **Ant (`build.xml`)** – Primary consumer; uses `<propertyfile>` or `<property>` tasks to load entries.  
- **Maven (`pom.xml`)** – May reference the same directories for system‑scoped dependencies (not typical).  
- **Environment Scripts** – `mnaas_dev.properties` and `mnaas_prod.properties` may override or extend these paths for dev vs. prod builds.  
- **Job Submission Scripts** – Runtime jobs inherit the classpath assembled from these entries.

# Operational Risks
- **Hard‑coded absolute paths** – Breaks on node re‑deployment or OS changes. *Mitigation:* Parameterize base directory via environment variable.  
- **Path drift** – Library versions may be upgraded without updating the file, causing `ClassNotFoundException`. *Mitigation:* Automate validation against the CDH parcel version.  
- **Missing directories** – Build fails if any path does not exist. *Mitigation:* Add pre‑build existence checks.  
- **Security exposure** – Absolute paths reveal internal filesystem layout. *Mitigation:* Restrict file read permissions to build service accounts.

# Usage
```bash
# Example: compile ingestion module using Ant
cd move-mediation-ingest/mnaas
ant -f build.xml compile -Dprops.file=bin/META-INF/maven/com.tcl.move/mnaas/mnaas.properties

# Debug: print resolved classpath
ant -f build.xml -verbose
```

# Configuration
- **File:** `move-mediation-ingest/mnaas/bin/META-INF/maven/com.tcl.move/mnaas/mnaas.properties` (current).  
- **Related Configs:** `mnaas_dev.properties`, `mnaas_prod.properties` (may reference or override entries).  
- **Environment Variables (optional):** `MNAAS_BASE_DIR` – can be used to prefix all paths if the file is refactored.

# Improvements
1. **Parameterize base directory:** Replace hard‑coded `/app/cloudera/parcels/CDH` with `${cdh.base}` resolved from an environment variable or a higher‑level properties file.  
2. **Add validation target:** Introduce an Ant `<target>` that verifies existence and version compatibility of each directory before compilation.