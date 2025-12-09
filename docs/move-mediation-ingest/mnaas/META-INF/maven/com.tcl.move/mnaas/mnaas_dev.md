# Summary
`mnaas_dev.properties` is a Maven‑generated properties file that enumerates absolute filesystem locations of all third‑party Hadoop ecosystem JAR directories required by the Move‑Mediation‑Ingest (MNAAS) Java utilities in a development environment. Production jobs load this file to build the Hadoop `Configuration` and to populate the `-libjars` argument for MapReduce, Hive, HBase, and generic loader jobs, ensuring the correct class‑path is supplied at runtime.

# Key Components
- **`hadoop.lib`** – Core Hadoop client JAR directory.  
- **`hive.lib`** – Hive execution engine JAR directory.  
- **`hbase.lib`** – HBase client JAR directory.  
- **`mapreduce.lib`** – Hadoop MapReduce runtime JAR directory.  
- **`hdfs.lib`** – Hadoop HDFS client JAR directory.  
- **`hadoopcli.lib`** – Hadoop command‑line utilities JAR directory.  
- **`generic.lib`** – Path to organization‑wide generic utility JARs.  
- **`mnaas.generic.lib`** – Path to MNAAS‑specific generic JARs (raw aggregation tables processing).  
- **`mnaas.generic.load`** – Path to MNAAS loader JARs required for data ingestion.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Job launch | `mnaas_dev.properties` (via classpath) | Java utilities read properties to resolve library directories; construct `-libjars` list for Hadoop commands (`hadoop jar`, `hive -f`, `hbase ...`). | Correct class‑path for Hadoop ecosystem; no direct data transformation. |
| Runtime | Hadoop/Hive/HBase services | Jobs execute using the supplied JARs. | Generated logs, processed data files, Hive tables, HBase rows. |
| Cleanup | N/A | No state persisted by this file. | None. |

# Integrations
- **MNAAS Java utilities** (`SequanceValidation`, `telene_file_stats`, etc.) load this file via `java.util.Properties` or Maven resource injection to obtain library paths.
- **Hadoop command wrappers** (`hadoop`, `hive`, `hbase`) receive the `-libjars` argument constructed from these entries.
- **Cluster configuration scripts** may substitute `mnaas_dev.properties` with `mnaas.properties` for production environments.

# Operational Risks
- **Path drift** – Hard‑coded absolute paths become stale after cluster upgrades or relocations. *Mitigation*: Automate path generation via environment variables or symlinks; validate existence at startup.
- **Missing JARs** – If any directory is missing or empty, jobs will fail with `ClassNotFoundException`. *Mitigation*: Include health‑check script that verifies each path contains expected core JARs before job submission.
- **Environment mismatch** – Using dev properties in production can cause version incompatibility. *Mitigation*: Enforce environment‑specific property file selection via CI/CD gating.

# Usage
```bash
# Example: launch SequanceValidation with dev libraries
export MNAAS_PROP=move-mediation-ingest/mnaas/META-INF/maven/com.tcl.move/mnaas/mnaas_dev.properties
java -cp $(cat $MNAAS_PROP | awk -F= '/\.lib/ {print $2}' | tr '\n' ':') \
     com.tcl.mnass.validation.SequanceValidation \
     --currentFile /data/incoming/20251208_sms.txt \
     --processedList /var/log/mnaas/processed.lst \
     --missingList   /var/log/mnaas/missing.lst
```
*Debug*: Verify constructed classpath contains expected JARs; run `java -verbose:class` to confirm loading.

# Configuration
- **File**: `move-mediation-ingest/mnaas/META-INF/maven/com.tcl.move/mnaas/mnaas_dev.properties`
- **Environment variables** (optional): `MNAAS_PROP` to point to alternative property file.
- **Dependent config**: `mnaas.properties` (production counterpart) may override these entries.

# Improvements
1. **Parameterize paths** – Replace absolute paths with placeholders resolved from environment variables (e.g., `${HADOOP_HOME}/lib`) to simplify upgrades and support multiple clusters.
2. **Add checksum validation** – Include a checksum manifest for each library directory and implement a startup verification step to detect corrupted or missing JARs early.