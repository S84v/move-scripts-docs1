# Summary
Defines runtime constants for the **MNAAS_GBS_Load** batch job. It imports shared common properties and specifies the locations of the GBS loader JAR, the properties directory, and the Java log file. These constants are consumed by the `MNAAS_GBS_Load.sh` script to execute a Java‑based load of GBS data into HDFS/Hive in the Move‑Mediation production environment.

# Key Components
- `GBSLoaderJarPath` – Absolute HDFS‑accessible path to `GBSLoader.jar`, the executable JAR that performs the GBS data load.  
- `MNAASPropertiesPath` – Directory containing ancillary property files required by the loader (e.g., DB connection, Hive table mappings).  
- `GBSLoaderJavaLogPath` – Local filesystem path for the Java process log (`MoveGBS.log`).  
- `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – Source statement that injects shared constants (e.g., Hadoop user, environment flags, Hive metastore URIs).

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|-------------|--------|--------------|
| 1. Shell script start | `MNAAS_GBS_Load.sh` sources this file | Environment variables are exported to the shell | Variables available to subsequent commands | None |
| 2. Java execution | `GBSLoaderJarPath` + `MNAASPropertiesPath` (property files) | `java -jar $GBSLoaderJarPath` reads ancillary properties, connects to source DB via Sqoop‑like logic, writes transformed data to HDFS and registers Hive tables | Data staged in HDFS (`/user/.../gbs/...`) and Hive tables populated | HDFS write, Hive metastore update |
| 3. Logging | `GBSLoaderJavaLogPath` | Java process redirects stdout/stderr to the log file | `MoveGBS.log` contains execution trace, errors, and metrics | Log file growth; requires rotation |

External services: Hadoop HDFS, Hive Metastore, source relational DB (via JDBC), optional Kerberos authentication (inherited from common properties).

# Integrations
- **MNAAS_CommonProperties.properties** – Provides global settings (e.g., `HADOOP_USER`, `HIVE_DB`, `SQOOP_JDBC_URL`).  
- **MNAAS_GBS_Load.sh** – Shell wrapper that sources this file, constructs the Java command, and handles exit codes.  
- **GBSLoader.jar** – Java application that implements the actual extraction, transformation, and load (ETL) logic.  
- **HDFS/Hive** – Destination layers where the processed GBS data resides.  

# Operational Risks
- **Path misconfiguration** – Incorrect absolute paths cause job failure; mitigate with CI linting of property files and automated existence checks before job start.  
- **Log file overflow** – Unbounded `MoveGBS.log` can exhaust disk; mitigate by implementing log rotation (logrotate or Hadoop log management).  
- **Dependency drift** – Changes in `MNAAS_CommonProperties.properties` may break expectations; mitigate by version‑pinning the common file and running regression tests on each release.  
- **Insufficient permissions** – Hadoop user may lack write access to HDFS target directories; mitigate by validating ACLs during deployment.

# Usage
```bash
# From the job scheduler or manually:
cd /app/hadoop_users/MNAAS/scripts
source ../config/MNAAS_GBS_Load.properties   # loads variables
java -jar "$GBSLoaderJarPath" \
     --props "$MNAASPropertiesPath/gbs_loader.conf" \
     >> "$GBSLoaderJavaLogPath" 2>&1
```
To debug, set `set -x` in the wrapper script and inspect `MoveGBS.log` for stack traces.

# Configuration
- **Environment Variables** (optional, may be overridden by common properties): `HADOOP_CONF_DIR`, `JAVA_HOME`.  
- **Referenced Config Files**:  
  - `MNAAS_CommonProperties.properties` (shared constants).  
  - Additional loader‑specific property files located under `$MNAASPropertiesPath` (e.g., `gbs_loader.conf`).  

# Improvements
1. **Parameterize log rotation** – Add a property `GBSLoaderLogRetentionDays` and integrate with a log‑rotation script to prevent disk saturation.  
2. **Validate paths at load time** – Introduce a small shell function that checks existence and permissions of `$GBSLoaderJarPath`, `$MNAASPropertiesPath`, and `$GBSLoaderJavaLogPath` before invoking the Java process; abort with clear error codes if validation fails.