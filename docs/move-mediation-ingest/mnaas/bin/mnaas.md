# Summary
The `mnaas.properties` file supplies absolute filesystem locations for Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop‑CLI, generic, and MNAAS‑specific JAR directories. It is read by the Ant build script (`build.xml`) and other deployment tooling to construct compile‑time and runtime classpaths for the MOVE‑Mediation Ingest (MNAAS) production pipeline.

# Key Components
- **hadoop.lib** – Path to Hadoop core libraries.  
- **hive.lib** – Path to Hive client libraries.  
- **hbase.lib** – Path to HBase client libraries.  
- **mapreduce.lib** – Path to Hadoop MapReduce libraries.  
- **hdfs.lib** – Path to Hadoop HDFS libraries.  
- **hadoopcli.lib** – Path to Hadoop CLI client libraries.  
- **generic.lib** – Path to organization‑wide generic JARs.  
- **mnaas.generic.lib** – Path to MNAAS‑specific processing JARs (raw aggregation).  
- **mnaas.generic.load** – Path to MNAAS‑specific loader JARs.

# Data Flow
- **Input:** Ant build script (`build.xml`) loads this file via `<property file="mnaas.properties"/>`.  
- **Processing:** Property values are concatenated into `classpath` attributes for `<javac>`, `<jar>`, and `<java>` tasks.  
- **Output:** Generated classpath strings used for compilation, packaging, and runtime execution of the MNAAS ingestion job.  
- **Side Effects:** None; file is read‑only at build time.  
- **External Services/DBs/Queues:** Not directly referenced; indirect interaction occurs when the built JAR accesses Hadoop/HBase/Hive services at runtime.

# Integrations
- **build.xml (Ant):** Primary consumer; builds compile‑time and runtime classpaths.  
- **pom.xml (Maven):** May reference the same directories for manual overrides, though Maven typically resolves dependencies from repositories.  
- **Deployment scripts:** Any wrapper that sets `ANT_OPTS` or invokes Ant with `-propertyfile mnaas.properties`.  
- **Runtime environment:** The resulting JAR expects the same directory layout on target nodes for successful execution.

# Operational Risks
- **Hard‑coded absolute paths:** Breaks on node re‑provisioning or OS upgrades. *Mitigation:* Use symlinks or environment‑variable substitution.  
- **Missing or outdated JARs:** Build failures or runtime `ClassNotFoundException`. *Mitigation:* Implement a validation step in the Ant script to verify existence of each path.  
- **Permission issues:** Non‑readable directories cause silent Ant failures. *Mitigation:* Enforce uniform filesystem ACLs for the service account.  
- **Path drift between dev and prod:** Inconsistent builds. *Mitigation:* Maintain separate, version‑controlled property files per environment and enforce CI checks.

# Usage
```bash
# Compile and package using the production property file
cd move-mediation-ingest/mnaas/bin
ant -propertyfile mnaas.properties clean compile jar

# Debug classpath resolution
ant -propertyfile mnaas.properties -verbose compile
```

# Configuration
- **File:** `move-mediation-ingest/mnaas/bin/mnaas.properties` (or environment‑specific variants such as `mnaas_dev.properties`).  
- **Environment Variables:** None required; all values are absolute paths defined in the file.  
- **Referenced Configs:** `build.xml` (Ant) reads this file; optional overrides via command‑line `-Dproperty=value`.

# Improvements
1. **Parameterize paths:** Replace absolute literals with `${HADOOP_HOME}`‑style variables resolved from environment variables or a central configuration service.  
2. **Add validation target:** Introduce an Ant `<target name="validate-paths">` that checks each directory exists and logs missing entries before compilation.