# Summary
`mnaas.properties` defines absolute filesystem paths for all third‑party and internal JAR libraries required by the MOVE‑Mediation Ingest (MNAAS) Ant build. The properties are consumed by `build.xml` to construct the classpath for compilation, packaging, and runtime execution of the MNAAS ingestion pipeline in the telecom production move system.

# Key Components
- **hadoop.lib** – Base Hadoop core JAR directory.  
- **hive.lib** – Hive client JAR directory.  
- **hbase.lib** – HBase client JAR directory.  
- **mapreduce.lib** – Hadoop MapReduce library directory.  
- **hdfs.lib** – HDFS client JAR directory.  
- **hadoopcli.lib** – Legacy Hadoop CLI JAR directory.  
- **generic.lib** – Directory for generic utility JARs shared across projects.  
- **mnaas.generic.lib** – Path to MNAAS‑specific utility JARs (e.g., raw‑aggregation table processors).  
- **mnaas.generic.load** – Path to loader JARs used by MNAAS for bulk data loading.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| Ant build (compile) | Java source (`src/`) | Classpath assembled from the properties | Compiled `.class` files in `build.dir` | None |
| Ant build (jar) | Compiled classes + resources | JAR packaging using classpath | Deployable `mnaas.jar` in `dist.dir` | None |
| Runtime (MNAAS job) | Input data files (HDFS, Kafka, etc.) | Job code loads libraries via classpath | Processed move‑mediation records (HDFS tables, Hive tables) | Writes to HDFS/Hive, may invoke external loaders |

# Integrations
- **`build.xml`** – References each property to build the Ant classpath (`<path>` elements).  
- **Hadoop ecosystem** – Libraries provide APIs for HDFS, MapReduce, Hive, and HBase interactions used by MNAAS jobs.  
- **MNAAS generic JARs** – Contain custom ingestion logic (`Raw_Aggr_tables_processing`, `Loader_Jars`) that integrate with the core pipeline.  
- **External scripts** – Scheduler (e.g., Oozie, Airflow) launches the generated JAR; they rely on the same library locations.

# Operational Risks
- **Hard‑coded absolute paths** – Break if CDH installation moves or version changes. *Mitigation*: Parameterize via environment variables or relative paths.  
- **Version drift** – Library directories may contain multiple versions; ambiguous class loading can cause runtime `NoSuchMethodError`. *Mitigation*: Pin exact JAR versions in a dedicated lib directory.  
- **Missing directories** – Build fails if any path is unavailable. *Mitigation*: Add pre‑build validation step.  
- **Security exposure** – Paths point to system‑wide locations; unauthorized modifications could affect the pipeline. *Mitigation*: Restrict filesystem permissions to the service account.

# Usage
```bash
# From the mnaas project root
ant -f build.xml clean compile jar
# Ant will read mnaas.properties automatically (property file is on the classpath)
# To debug classpath:
ant -f build.xml -Dverbose=true
```
For runtime debugging, set Java system property `java.library.path` to include the same directories or export `CLASSPATH` using the property values.

# Configuration
- **File**: `move-mediation-ingest/mnaas/mnaas.properties` (must be on Ant’s classpath).  
- **Referenced by**: `build.xml` via `<property file="mnaas.properties"/>`.  
- **No environment variables** are required, but the paths assume the following installations:
  - CDH at `/app/cloudera/parcels/CDH/`
  - Custom JARs at `/app/hadoop_users/…`

# Improvements
1. **Externalize paths** – Replace absolute paths with environment‑variable placeholders (e.g., `${CDH_HOME}`) and document required env vars.  
2. **Add checksum validation** – Include a script that verifies the existence and integrity (SHA‑256) of each referenced JAR before the Ant build runs.