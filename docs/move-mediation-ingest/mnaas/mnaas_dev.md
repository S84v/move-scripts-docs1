# Summary
`mnaas_dev.properties` supplies absolute filesystem paths for all third‑party and internal JAR libraries required by the MOVE‑Mediation Ingest (MNAAS) Ant build in a development environment. The properties are consumed by `build.xml` to construct the classpath for compilation, packaging, and runtime execution of the MNAAS ingestion pipeline.

# Key Components
- **hadoop.lib** – Base Hadoop core JAR directory.  
- **hive.lib** – Hive client JAR directory.  
- **hbase.lib** – HBase client JAR directory.  
- **mapreduce.lib** – Hadoop MapReduce JAR directory.  
- **hdfs.lib** – Hadoop HDFS JAR directory.  
- **hadoopcli.lib** – Hadoop command‑line client JAR directory.  
- **generic.lib** – Directory for generic utility JARs shared across projects.  
- **mnaas.generic.lib** – Directory for MNAAS‑specific generic JARs (e.g., raw aggregation tables processing).  
- **mnaas.generic.load** – Directory for MNAAS loader JARs.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| Ant build (compile) | Java source files (`src.dir`) | Classpath assembled from the above properties | Compiled `.class` files in `build.dir` | None |
| Ant build (package) | Compiled classes + resources | JAR creation using classpath | `dist.dir` JAR | None |
| Runtime execution | JAR + external services (Hadoop, Hive, HBase) | Java process launched with classpath derived from properties | Processed move‑mediation records (e.g., DB writes, Kafka messages) | Network I/O to Hadoop ecosystem |

# Integrations
- **`build.xml`**: References each property to build the Ant classpath (`<path>` elements).  
- **`mnaas.properties`** (production counterpart): Same keys, different values for production paths.  
- **Hadoop ecosystem**: Paths point to CDH installation directories; the built JAR interacts with HDFS, Hive, HBase, and MapReduce jobs.  
- **MNAAS loader modules**: JARs in `mnaas.generic.load` are loaded at runtime for data ingestion.

# Operational Risks
- **Hard‑coded absolute paths** – Break if CDH is re‑installed or moved. *Mitigation*: Parameterize base directory via environment variable or relative path.  
- **Environment drift** – Development and production may diverge, causing classpath mismatches. *Mitigation*: Maintain a single source of truth (e.g., Maven/Gradle) and generate properties automatically.  
- **Missing JARs** – If any directory is empty or JARs are removed, Ant build fails. *Mitigation*: Add validation targets in `build.xml` to verify existence before compilation.  

# Usage
```bash
# Set ANT_OPTS if needed, then invoke Ant with the dev properties
cd move-mediation-ingest/mnaas
ant -propertyfile mnaas_dev.properties clean compile jar
# To debug classpath issues
ant -verbose -propertyfile mnaas_dev.properties compile
```

# Configuration
- **File**: `move-mediation-ingest/mnaas/mnaas_dev.properties` (development).  
- **Related files**: `move-mediation-ingest/mnaas/mnaas.properties` (production).  
- **Environment variables** (optional, if modified): None required by default; paths are absolute.  

# Improvements
1. **Externalize base directory** – Introduce a single property (e.g., `cdh.base.dir`) and derive all Hadoop‑related paths via `${cdh.base.dir}/...` to simplify upgrades.  
2. **Validate at build time** – Add an Ant target that checks each directory for required JARs and fails fast with a clear error message.