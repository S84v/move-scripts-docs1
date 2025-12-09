# Summary
`mnaas.properties` is a Maven‑generated resource that defines the class‑path locations for all third‑party Hadoop ecosystem libraries required by the MNAAS (Move‑Mediation‑Ingest) Java utilities at runtime. Production jobs load this file to construct the Hadoop `Configuration` and to populate the `-libjars` argument for MapReduce, Hive, HBase, and generic loader JARs used throughout the mediation pipeline.

# Key Components
- **`hadoop.lib`** – Core Hadoop client JAR directory.  
- **`hive.lib`** – Hive execution engine JAR directory.  
- **`hbase.lib`** – HBase client JAR directory.  
- **`mapreduce.lib`** – Hadoop MapReduce library directory.  
- **`hdfs.lib`** – HDFS client JAR directory.  
- **`hadoopcli.lib`** – Legacy Hadoop CLI JAR directory (client‑0.20).  
- **`generic.lib`** – Path to a shared “Generic” JAR repository for custom utilities.  
- **`mnaas.generic.lib`** – Path to MNAAS‑specific generic JARs (raw aggregation tables processing).  
- **`mnaas.generic.load`** – Path to MNAAS loader JARs used by ingestion jobs.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input** | Read | The properties file is read by Java utilities (e.g., `Seqno_validation`, `SequanceValidation`, `telene_file_stats`) during startup. |
| **Output** | In‑memory configuration | Values are injected into `org.apache.hadoop.conf.Configuration` or concatenated into a `-libjars` command‑line argument for Hadoop/YARN job submission. |
| **Side Effects** | Class‑path augmentation | Enables runtime loading of native Hadoop/Hive/HBase classes without bundling them in the application JAR. |
| **External Services** | None directly; indirect via libraries (HDFS, Hive Metastore, HBase). |

# Integrations
- **MNAAS command‑line utilities** (`Seqno_validation`, `SequanceValidation`, `telene_file_stats`) invoke `PropertiesLoader` (or equivalent) to retrieve library paths and configure Hadoop jobs.
- **Maven build** packages this file under `META-INF/maven/...` so that the compiled JAR contains the runtime class‑path metadata.
- **YARN / MapReduce launcher** reads the generated `-libjars` list to distribute required JARs across the cluster nodes.
- **Hive/HBase clients** rely on the paths to locate connector JARs for metadata queries and table writes.

# Operational Risks
- **Path drift** – Hard‑coded absolute paths may become invalid after OS upgrades or CDH re‑deployment. *Mitigation*: Use symbolic links or environment‑driven base directories; validate paths at job start.
- **Version mismatch** – Library directories may contain multiple versions; incorrect JARs could cause `NoSuchMethodError`. *Mitigation*: Pin exact JAR filenames or use a version‑specific sub‑directory.
- **Security exposure** – Unrestricted read access to `/app/hadoop_users/*` could allow malicious JAR injection. *Mitigation*: Enforce file‑system ACLs and checksum verification of JARs during deployment.

# Usage
```bash
# Example: launching Seqno_validation with libjars derived from mnaas.properties
java -cp $(cat mnaas.properties | grep '\.lib=' | cut -d= -f2 | tr '\n' ':' ) \
     com.tcl.mnass.validation.Seqno_validation \
     --currentFile /data/incoming/2025/12/08/file_20251208_001.dat \
     --processedList /var/mnaas/processed.lst \
     --missingList   /var/mnaas/missing.lst
```
*Debug*: Print resolved libjars:
```bash
grep '\.lib=' mnaas.properties | awk -F= '{print $2}' | tr '\n' ':' 
```

# Configuration
- **File location**: `move-mediation-ingest/mnaas/META-INF/maven/com.tcl.move/mnaas/mnaas.properties` (packaged inside the MNAAS JAR).  
- **Environment variables** (optional overrides):
  - `MNAAS_LIB_ROOT` – base directory to prepend to each relative path if the deployment uses a different root.
- **Referenced config files**: None; all values are self‑contained.

# Improvements
1. **Parameterize base directory** – Replace absolute prefixes with `${MNAAS_LIB_ROOT}` placeholders and resolve at runtime to simplify migrations across environments.  
2. **Add checksum manifest** – Include SHA‑256 hashes for each library directory/JAR to enable integrity verification before job submission.