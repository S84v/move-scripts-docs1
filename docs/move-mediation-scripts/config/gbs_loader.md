# Summary
`gbs_loader.properties` supplies environment‑specific configuration for the GBS (Global Billing System) loader component of the Move‑Mediation pipeline. It imports shared MNAAS properties and defines the absolute locations of the loader JAR and its Java‑level log file, enabling downstream scripts to invoke the GBS loader consistently across environments.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  *Loads common MNAAS variables (e.g., `MNAASJarPath`, `MNAASLocalPath`).*
- **GBSLoaderJarPath** – `$MNAASJarPath/Loader_Jars/GBSLoader.jar`  
  *Full filesystem path to the executable JAR for the GBS loader.*
- **GBSLoaderJavaLogPath** – `$MNAASLocalPath/MNAASCronLogs/MoveGBS.log`  
  *Destination file for Java‑level logging produced by the loader.*

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| Script initialization | `MNAAS_CommonProperties.properties` | Source variables into current shell environment | Populated env vars (`MNAASJarPath`, `MNAASLocalPath`) | None |
| Loader invocation (by wrapper script) | `GBSLoaderJarPath` (JAR) + runtime arguments (e.g., date range) | `java -jar $GBSLoaderJarPath …` | Data loaded into Hive/Impala tables (time_dimension, dateinterval, etc.) | Writes to `$GBSLoaderJavaLogPath`; may generate HDFS files, Hive partitions |

# Integrations
- **MNAAS_CommonProperties.properties** – central repository of base paths, Hadoop configuration, and cluster credentials.  
- **Wrapper shell scripts** (e.g., `GBSLoader.sh`) source this file to obtain the JAR location and log path.  
- **Hive/Impala** – the loader JAR contains Spark/MapReduce jobs that write to Hive dimension tables.  
- **HDFS** – loader may read source billing files from HDFS directories defined elsewhere in common properties.  

# Operational Risks
- **Missing or outdated common properties** → undefined `MNAASJarPath`/`MNAASLocalPath`; job fails at start. *Mitigation*: Validate existence of sourced file and required vars before use.  
- **Hard‑coded relative paths** → breakage when directory layout changes. *Mitigation*: Centralize path definitions in `MNAAS_CommonProperties.properties`.  
- **Log file permission issues** → loader cannot write to `MoveGBS.log`. *Mitigation*: Ensure cron user has write access to `$MNAASLocalPath/MNAASCronLogs`.  
- **JAR version drift** → incompatible schema changes. *Mitigation*: Pin JAR version via naming convention or checksum verification.

# Usage
```bash
# Load configuration into current shell
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/gbs_loader.properties

# Run the loader (typically invoked by a wrapper script)
java -jar "$GBSLoaderJarPath" --start-date 2025-09-01 --end-date 2025-09-30 >> "$GBSLoaderJavaLogPath" 2>&1
```
For debugging, echo the resolved variables:
```bash
echo "Jar: $GBSLoaderJarPath"
echo "Log: $GBSLoaderJavaLogPath"
```

# Configuration
- **Environment variables** (populated by `MNAAS_CommonProperties.properties`):  
  - `MNAASJarPath` – base directory for all loader JARs.  
  - `MNAASLocalPath` – base directory for local logs and temporary files.  
- **Referenced config file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`.

# Improvements
1. **Add validation block** to verify that `GBSLoaderJarPath` exists and is executable; abort with clear error if not.
2. **Externalize log rotation** by referencing a log‑rotation policy file or using a timestamped log name (e.g., `MoveGBS_$(date +%Y%m%d).log`).