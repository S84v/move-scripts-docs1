# Summary
`mnaas_prod.properties` defines absolute filesystem locations for Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop‑CLI, generic, and MNAAS‑specific JAR directories used by the MOVE‑Mediation Ingest (MNAAS) production pipeline. The Ant build script (`build.xml`) reads this file to assemble compile‑time and runtime classpaths for compiling, packaging, and executing ingestion jobs in the production environment.

# Key Components
- **hadoop.lib** – Hadoop core library directory.  
- **hive.lib** – Hive client library directory.  
- **hbase.lib** – HBase client library directory.  
- **mapreduce.lib** – Hadoop MapReduce library directory.  
- **hdfs.lib** – Hadoop HDFS library directory.  
- **hadoopcli.lib** – Hadoop CLI client library directory.  
- **generic.lib** – Directory containing generic third‑party JARs.  
- **mnaas.generic.lib** – Directory with MNAAS‑specific processing JARs.  
- **mnaas.generic.load** – Directory with MNAAS loader JARs.

# Data Flow
- **Input:** Ant build (`build.xml`) loads this properties file via `<property file=".../mnaas_prod.properties"/>`.  
- **Processing:** Property values are concatenated to form the compile‑time and runtime classpaths.  
- **Output:** Generated classpath strings passed to `javac`, `jar`, and `java` tasks; ultimately used by ingestion jobs executed on the Hadoop cluster.  
- **Side Effects:** None beyond classpath construction.  
- **External Services/DBs/Queues:** Not directly accessed; indirect interaction occurs when compiled jobs submit MapReduce/Hive/HBase tasks to the Hadoop ecosystem.

# Integrations
- **Ant build script (`build.xml`)** – primary consumer; references each property to build classpaths.  
- **Deployment scripts** – may copy or symlink the JARs referenced here to cluster nodes.  
- **Runtime job launchers** – rely on the same classpath definitions when invoking the MNAAS main class or Hadoop jobs.

# Operational Risks
- **Hard‑coded absolute paths** – break if CDH installation moves or version changes. *Mitigation:* validate paths at build start; maintain version‑specific property files.  
- **Missing or outdated JARs** – cause compilation or runtime failures. *Mitigation:* automate verification of directory contents via CI checks.  
- **Permission issues** – build agents may lack read access to the directories. *Mitigation:* enforce consistent filesystem ACLs for the build user.  
- **Environment drift** between dev and prod properties may lead to inconsistent behavior. *Mitigation:* enforce property‑file parity checks in CI.

# Usage
```bash
# From the mnaas/bin directory
ant -propertyfile mnaas_prod.properties compile jar run
```
- `compile` – compiles Java sources using classpath built from the properties.  
- `jar` – packages compiled classes into `mnaas.jar`.  
- `run` – executes the main class with the runtime classpath.

# Configuration
- **File:** `mnaas_prod.properties` (this file).  
- **Referenced by:** `build.xml` via `<property file="${basedir}/mnaas_prod.properties"/>`.  
- **No environment variables** are required; all paths are absolute.

# Improvements
1. **Parameterize base directory** – introduce a `${cdh.base}` property and construct sub‑paths relative to it to simplify upgrades.  
2. **Add validation target** – Ant target that checks existence and readability of each directory/JAR before compilation, aborting early on misconfiguration.