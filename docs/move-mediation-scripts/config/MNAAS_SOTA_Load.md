# Summary
`MNAAS_SOTA_Load.properties` is a Bash‑sourced configuration fragment that supplies runtime constants for the “SOTA Load” step of the MNAAS daily‑processing pipeline. It imports shared common properties and defines paths for logs, status files, Hive/Impala tables, HDFS locations, Sqoop parameters, and the Java loader JAR used by `MNAAS_Load_SOTA_Table.sh`.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – loads global variables.  
- **Sqoop variables** – `cust_master_SqoopLog`, `cust_HDFSSqoopDir`, `cust_master_tblname` – used by Sqoop import of `customer_master_temp`.  
- **Process metadata** – `sota_table_load_ProcessStatusFileName`, `sota_load_Scriptname`, `sota_load_LogPath` – status tracking and logging for the load script.  
- **Data file references** – `sota_load_file`, `sota_load_prop_file` – CSV source and Java batch properties.  
- **Hive/Impala table names** – `dbname`, `api_best_window_sota_temp_tbl`, `api_best_window_sota_tbl`, `pgw_best_window_pred_tbl`.  
- **Java loader** – `sota_load_jar` – path to `BestWindow.jar` executed by the load script.  
- **HDFS enrichment paths** – `hdfs_enrich_path`, `hdfs_enrich_file` – locations for enriched SOTA data before Hive load.

# Data Flow
| Stage | Input | Transformation | Output |
|-------|-------|----------------|--------|
| Sqoop import | `customer_master_temp` (source DB) | Sqoop writes to HDFS | `$cust_HDFSSqoopDir` |
| CSV enrichment | `$sota_load_file` (local CSV) | Java `BestWindow.jar` processes → enriched CSV | `$hdfs_enrich_file` |
| Hive load | Enriched CSV on HDFS | `LOAD DATA` or `INSERT OVERWRITE` into `$api_best_window_sota_tbl` (and temp table) | Hive tables populated |
| Status & logging | Script execution | Writes to `$sota_table_load_ProcessStatusFileName` and `$sota_load_LogPath` | Process audit records |

Side effects: creation of HDFS directories/files, Hive table partitions, log files, and status file updates.

External services: Hadoop HDFS, Hive/Impala, Sqoop, Java runtime, underlying relational source for Sqoop.

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base paths (`MNAASLocalLogPath`, `MNAASConfPath`, etc.).  
- **`MNAAS_Load_SOTA_Table.sh`** – consumes all variables defined here to orchestrate Sqoop, Java JAR execution, and Hive load.  
- **`MNAAS_Java_Batch.properties`** – passed to the Java JAR for batch configuration.  
- **Sqoop** – invoked by the shell script using `cust_master_*` variables.  
- **Hive/Impala** – accessed via JDBC/Beeline within the shell script using table names.  

# Operational Risks
- **Path mismatch** – hard‑coded absolute paths may become stale after filesystem re‑layout. *Mitigation*: centralize base directories in common properties and validate at script start.  
- **Jar version drift** – `BestWindow.jar` updates without corresponding property change may cause incompatibility. *Mitigation*: version‑stamp JAR filename and lock via CI.  
- **Log/Status file contention** – concurrent runs could overwrite status file. *Mitigation*: include timestamp or run‑ID in status filename.  
- **HDFS permission errors** – script may lack write permission on `$hdfs_enrich_path`. *Mitigation*: enforce ACLs and pre‑run permission check.  

# Usage
```bash
# Load common properties first
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this config
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SOTA_Load.properties

# Execute the load script (debug mode optional)
set -x   # enable Bash tracing if needed
$MNAASShellScriptPath/MNAAS_Load_SOTA_Table.sh
```
To debug, uncomment `setparameter='set -x'` in the config or prepend `bash -x` to the script call.

# Configuration
- **Environment variables** injected by `MNAAS_CommonProperties.properties`:  
  - `MNAASLocalLogPath`  
  - `MNAASConfPath`  
  - `MNAASShellScriptPath`  
  - `MNAASPropertiesPath`  
- **Referenced files**:  
  - `MNAAS_Java_Batch.properties` (Java batch config)  
  - `sota_enriched_data.csv` (input CSV)  
  - `BestWindow.jar` (Java loader)  

# Improvements
1. **Parameterize base directories** – replace absolute paths with placeholders resolved at runtime to simplify environment migration.  
2. **Add validation block** – script should verify existence and write permission of all referenced HDFS and local paths before proceeding, exiting with clear error codes.