# Summary
`mnaas_prod.properties` is a Maven‑generated properties file used in the production environment of the Move‑Mediation‑Ingest (MNAAS) pipeline. It enumerates absolute filesystem locations of all third‑party Hadoop ecosystem JAR directories required at runtime. The file is read by MNAAS utilities to construct a Hadoop `Configuration` and to populate the `-libjars` argument for MapReduce, Hive, HBase, and generic loader jobs, ensuring the correct class‑path for production jobs.

# Key Components
- **`hadoop.lib`** – Path to core Hadoop client libraries.  
- **`hive.lib`** – Path to Hive client libraries.  
- **`hbase.lib`** – Path to HBase client libraries.  
- **`mapreduce.lib`** – Path to Hadoop MapReduce libraries.  
- **`hdfs.lib`** – Path to HDFS client libraries.  
- **`hadoopcli.lib`** – Path to legacy Hadoop CLI client JARs.  
- **`generic.lib`** – Path to generic third‑party JARs used across pipelines.  
- **`mnaas.generic.lib`** – Path to MNAAS‑specific generic JARs (e.g., raw aggregation tables processing).  
- **`mnaas.generic.load`** – Path to MNAAS loader JARs required for data ingestion jobs.

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | Production job scripts (Java utilities, shell wrappers) read this properties file at startup. |
| **Processing** | Values are parsed and concatenated into a comma‑separated list for the Hadoop `-libjars` command‑line option; also used to set `Configuration` class‑path entries. |
| **Output** | No direct output; influences the class‑path of launched Hadoop jobs, enabling them to locate required libraries. |
| **Side Effects** | Incorrect paths cause `ClassNotFoundException` or job failures. |
| **External Services** | Hadoop/YARN cluster, Hive Metastore, HBase, Impala – accessed by jobs that rely on the libraries listed. |

# Integrations
- **Java utilities** (e.g., `telene_file_stats`, other MNAAS command‑line tools) load this file via `java.util.Properties` to build runtime configurations.  
- **Shell wrappers / Oozie workflows** reference the file when constructing `-libjars` arguments for `hive`, `spark-submit`, or `mapred` commands.  
- **Maven build** includes the file in the JAR under `META-INF/maven/...` so it is packaged with the application artifact.  
- **Production deployment scripts** copy the file to the class‑path of each node in the cluster.

# Operational Risks
- **Path drift** – Absolute paths may become stale after Hadoop/CDH upgrades. *Mitigation*: Automate regeneration of the file during platform upgrades or use symbolic links.  
- **Missing JARs** – Deletion or permission changes on listed directories cause job failures. *Mitigation*: Periodic health‑check script that validates existence and readability of each path.  
- **Environment mismatch** – Using the production properties file in a dev environment leads to class‑path conflicts. *Mitigation*: Enforce environment‑specific naming (`mnaas_dev.properties`) and guard against accidental cross‑use.  
- **Security exposure** – Absolute filesystem locations may reveal internal directory structure. *Mitigation*: Restrict file read permissions to the service account only.

# Usage
```bash
# Example: launching a MapReduce job with the production libjars
LIBJARS=$(cat $MNAAS_HOME/META-INF/maven/com.tcl.move/mnaas/mnaas_prod.properties \
          | grep -v '^#' \
          | awk -F= '{print $2}' \
          | paste -sd, -)

hadoop jar myjob.jar com.tcl.move.mnaas.MyJob \
    -libjars "$LIBJARS" \
    -D mapreduce.job.queuename=production \
    -input /data/input \
    -output /data/output
```
*Debug*: Print the resolved libjars list and verify each path exists before job submission.

# Configuration
- **File location**: `move-mediation-ingest/mnaas/META-INF/maven/com.tcl.move/mnaas/mnaas_prod.properties` (packaged inside the MNAAS JAR).  
- **Referenced by**: Java utilities (`java.util.Properties.load`), shell scripts (`source`/`cat`), Oozie workflow XML (`<property>` tags).  
- **Environment variables**: `MNAAS_HOME` (root of the MNAAS installation) is commonly used to locate the file. No other env vars are required.

# Improvements
1. **Parameterize paths** – Replace hard‑coded absolute paths with environment variables or relative paths resolved at runtime to simplify upgrades and reduce maintenance.  
2. **Add checksum validation** – Include a SHA‑256 hash for each JAR directory to detect accidental corruption or unauthorized modifications during job startup.