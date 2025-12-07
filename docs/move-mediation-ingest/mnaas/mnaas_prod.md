# Summary
`mnaas_prod.properties` supplies absolute filesystem locations of Hadoop, Hive, HBase, MapReduce, HDFS, generic, and MNAAS‑specific JAR directories for the production deployment of the MOVE‑Mediation Ingest (MNAAS) module. The Ant build (`build.xml`) reads these properties to assemble the compile‑time and runtime classpaths required by the ingestion pipeline.

# Key Components
- **hadoop.lib** – Hadoop core library directory.  
- **hive.lib** – Hive client library directory.  
- **hbase.lib** – HBase client library directory.  
- **mapreduce.lib** – Hadoop MapReduce library directory.  
- **hdfs.lib** – HDFS library directory.  
- **hadoopcli.lib** – Hadoop client (0.20) library directory.  
- **generic.lib** – Shared generic JAR repository for internal utilities.  
- **mnaas.generic.lib** – MNAAS‑specific generic JAR repository (raw aggregation tables).  
- **mnaas.generic.load** – MNAAS loader JAR repository.

# Data Flow
- **Inputs:** Ant build (`build.xml`) invokes `${propertyfile}` to load the above key‑value pairs.  
- **Outputs:** Constructed classpath strings injected into `<javac>`, `<jar>`, and runtime `<java>` tasks.  
- **Side Effects:** None; purely declarative.  
- **External Services/DBs/Queues:** Not accessed directly; paths point to libraries that may interact with HDFS, Hive, HBase, and MapReduce at runtime.

# Integrations
- **build.xml** – References `${hadoop.lib}`, `${hive.lib}`, etc., to populate `<classpath>` elements for compilation and packaging.  
- **mnaas_dev.properties / mnaas.properties** – Parallel property files for dev or generic builds; selection controlled by Ant target (`-Denv=prod`).  
- **Runtime scripts** – Any wrapper that launches the MNAAS JAR (e.g., `run-mnaas.sh`) reads the same classpath variables to set `CLASSPATH`.

# Operational Risks
- **Hard‑coded absolute paths** – Break if CDH installation moves; mitigate with symlinks or environment‑driven base directory.  
- **Version drift** – Paths may point to outdated JARs; enforce version lock via checksum validation in CI.  
- **Missing directories** – Build fails silently if a directory is absent; add pre‑build existence checks.  
- **Security exposure** – Absolute paths reveal internal filesystem layout; restrict file permissions.

# Usage
```bash
# Compile and package for production
ant -propertyfile move-mediation-ingest/mnaas/mnaas_prod.properties -Denv=prod compile jar

# Run the assembled JAR (example wrapper)
./run-mnaas.sh -Dconfig=prod
```

# Configuration
- **File:** `move-mediation-ingest/mnaas/mnaas_prod.properties` (referenced via Ant `-propertyfile`).  
- **Environment Variables (optional):** `CDH_ROOT` can be used to construct paths if the build.xml is modified to prepend it.  
- **Related Configs:** `build.xml`, `mnaas_dev.properties`, `mnaas.properties`.

# Improvements
1. **Parameterize base directory:** Replace hard‑coded `/app/cloudera/parcels/CDH` with `${cdh.root}` derived from an environment variable or a separate `global.properties` file.  
2. **Add validation target:** Introduce an Ant `<target name="validate‑paths">` that checks existence and readability of each directory before compilation, failing fast with a clear error message.