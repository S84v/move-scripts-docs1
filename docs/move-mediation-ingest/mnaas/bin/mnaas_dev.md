# Summary
`mnaas_dev.properties` provides absolute filesystem locations for Hadoop‑related and MNAAS‑specific JAR directories. In the MOVE‑Mediation Ingest (MNAAS) production pipeline the Ant build script (`build.xml`) reads this file to assemble compile‑time and runtime classpaths for compiling, packaging, and executing the ingestion jobs.

# Key Components
- **hadoop.lib** – Hadoop core library directory.  
- **hive.lib** – Hive client library directory.  
- **hbase.lib** – HBase client library directory.  
- **mapreduce.lib** – Hadoop MapReduce library directory.  
- **hdfs.lib** – Hadoop HDFS library directory.  
- **hadoopcli.lib** – Hadoop command‑line utilities library directory.  
- **generic.lib** – Shared generic JAR repository for custom utilities.  
- **mnaas.generic.lib** – MNAAS‑specific processing JAR repository (raw aggregation).  
- **mnaas.generic.load** – MNAAS‑specific loader JAR repository.

# Data Flow
1. **Input** – Ant build (`build.xml`) loads `mnaas_dev.properties`.  
2. **Processing** – Property values are concatenated into classpath strings for `javac`, `jar`, and runtime execution tasks.  
3. **Output** – Generated classpath environment variables; compiled classes and packaged JARs.  
4. **Side Effects** – Failure to locate any listed directory aborts the build; successful builds produce artifacts ready for deployment to the production Hadoop cluster.  
5. **External Services** – None directly; indirect dependency on the underlying Hadoop/Hive/HBase installations referenced by the paths.

# Integrations
- **build.xml (Ant)** – Primary consumer; referenced via `<property file="mnaas_dev.properties"/>`.  
- **deployment scripts** – May source the same properties to set runtime `CLASSPATH` for job submission (e.g., `hadoop jar ...`).  
- **IDE build configurations** – Can import the file to mirror Ant classpaths for local development.

# Operational Risks
- **Hard‑coded absolute paths** – Break when cluster layout changes; mitigated by maintaining a version‑controlled copy per environment and synchronizing with Ops.  
- **Missing or inaccessible directories** – Causes build failure; mitigated by pre‑flight validation script.  
- **Permission mismatches** – Build user may lack read access; mitigated by enforcing group ownership (`hadoop_users`).  
- **Environment drift** – Different CDH versions may place libraries in alternate locations; mitigated by parameterizing base install path.

# Usage
```bash
# From the mnaas/bin directory
ant -propertyfile mnaas_dev.properties compile jar
# To debug classpath composition
ant -propertyfile mnaas_dev.properties -verbose compile
```

# Configuration
- **File** – `move-mediation-ingest/mnaas/bin/mnaas_dev.properties` (environment‑specific).  
- **No environment variables** are required; all paths are defined within the file.  
- **Related config** – `mnaas_prod.properties` for production deployment; same key set with production‑specific paths.

# Improvements
1. Replace absolute paths with a base variable (e.g., `cdh.base=/app/cloudera/opt/cloudera/parcels/CDH`) and derive sub‑paths to simplify migration across clusters.  
2. Add a validation target in `build.xml` that checks existence and readability of each directory, aborting early with a clear error report.