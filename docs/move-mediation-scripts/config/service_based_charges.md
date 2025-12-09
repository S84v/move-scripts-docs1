# Summary
`service_based_charges.properties` is a Bash‑style configuration file for the Move mediation pipeline’s “service‑based charges” job. It sources common MNAAS properties, defines job‑specific variables (log path, status file, class names, Hive table names, refresh statements, and script name), and optionally enables shell debugging. The file is sourced by the `service_based_charges.sh` driver before launching the Java/Hive job that materialises the `service_based_charges_table` view used in downstream billing reports.

# Key Components
- **MNAAS_CommonProperties.properties** – global property source (Hadoop config, credential paths, base log directory).  
- **setparameter** – toggles Bash debug mode (`set -x`).  
- **MNAAS_service_based_charges_LogPath** – absolute log file path with daily suffix.  
- **service_based_charges_ProcessStatusFileName** – HDFS/local file used for job status tracking.  
- **Dname_service_based_charges** – logical job identifier.  
- **service_based_charges_temp_classname / service_based_charges_classname** – fully‑qualified Java class names for the temporary and final Hive UDF/Mapper implementations.  
- **service_based_charges_temp_tblname / service_based_charges_tblname** – Hive table identifiers for staging and final results.  
- **service_based_charges_*_refresh** – Hive `REFRESH` statements constructed with `$dbname` to invalidate metadata caches after table writes.  
- **MNAAS_service_based_charges_tblname_ScriptName** – driver script name (`service_based_charges.sh`) that consumes this properties file.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1 | `MNAAS_CommonProperties.properties` (environment vars, Hadoop config) | Sourced into current shell | Variables populated | None |
| 2 | `service_based_charges.sh` reads this file | Substitutes variables into Hive/Java command line | Hive job execution (temp table → final table) | Writes to `$MNAAS_service_based_charges_LogPath`, updates `service_based_charges_ProcessStatusFileName` |
| 3 | Hive tables (`$service_based_charges_temp_tblname`, `$service_based_charges_tblname`) | `REFRESH` statements executed | Updated Hive metastore cache | Potential HDFS I/O |

External services: Hadoop/YARN cluster, Hive Metastore, HDFS for logs and status file.

# Integrations
- **runBARep.properties / runMLNS.properties** – share the same common properties source; job ordering may be orchestrated by higher‑level Bash wrappers.  
- **service_based_charges.sh** – primary driver that sources this file, builds the Java classpath, and invokes the `com.tcl.mnaas.billingviews.*` classes via `spark-submit` or `hive -e`.  
- **MNAAS_Property_Files/MNAAS_CommonProperties.properties** – provides `$MNAASLocalLogPath`, `$dbname`, Hadoop security tokens, and default queue settings.  
- **Billing downstream pipelines** – consume the final Hive table `service_based_charges_table` for invoice generation.

# Operational Risks
- **Missing common properties** – job fails early; mitigate with validation of source file existence.  
- **Log path permission errors** – logs not written, obscuring failures; ensure `$MNAASLocalLogPath` is writable by the job user.  
- **Stale Hive metadata** – if `REFRESH` statements are omitted or fail, downstream queries may read outdated data; enforce exit‑code checks after each `REFRESH`.  
- **Debug flag left enabled** – excessive console output may fill logs; default `setparameter` to empty and enable only for troubleshooting.

# Usage
```bash
# Source the configuration (usually done inside the driver script)
source /app/hadoop_users/MNAAS/MNAAS_Configuration_Files/service_based_charges.properties

# Optional: enable Bash tracing for debugging
eval "$setparameter"

# Execute the driver script
bash $MNAAS_service_based_charges_tblname_ScriptName
```
To debug, set `setparameter='set -x'` and re‑run; monitor `$MNAAS_service_based_charges_LogPath` for detailed logs.

# Configuration
- **Environment variables** injected by `MNAAS_CommonProperties.properties`:  
  - `MNAASLocalLogPath` – base directory for logs.  
  - `dbname` – Hive database name used in refresh statements.  
  - Hadoop/YARN settings (`HADOOP_CONF_DIR`, `YARN_QUEUE`, etc.).  
- **File references**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (source).  
  - `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/service_based_charges_ProcessStatusFile` (status tracking).  

# Improvements
1. **Parameter validation block** – add a Bash function that checks existence and readability of all required variables/files before job launch, exiting with a clear error code.  
2. **Centralised logging wrapper** – replace manual log‑path concatenation with a function that rotates logs and enforces a consistent naming scheme across all Move jobs.