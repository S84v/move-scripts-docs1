**High‑Level Documentation – `move-mediation-scripts/bin/runGBS.sh`**

---

### 1. Purpose (One‑Paragraph Summary)
`runGBS.sh` is a thin orchestration wrapper that launches the Geneva (GBS) data‑loader Java batch job for a specific billing period. It first sources the central **MNAAS Java Batch** property file to obtain environment‑specific variables (paths to JARs, log directories, and auxiliary property files). It then executes the `GevevaLoader` JAR with a hard‑coded month/year argument (`06-2020`) together with the MNAAS Java property path and a log‑file destination. The script is invoked as part of the nightly mediation pipeline to ingest raw GBS records into the MNAAS data‑warehouse for downstream analytics and reporting.

---

### 2. Key Elements & Responsibilities

| Element | Type | Responsibility |
|---------|------|-----------------|
| `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties` | Shell *source* | Loads all required environment variables (e.g., `GevevaLoaderJarPath`, `Mnaas_java_propertypath`, `GenevaLoaderJavaLogPath`). |
| `java -jar $GevevaLoaderJarPath 06-2020 $Mnaas_java_propertypath $GenevaLoaderJavaLogPath N` | Command | Executes the Geneva Loader Java batch (`$GevevaLoaderJarPath`). Arguments: <br>1. Billing period (`06-2020`). <br>2. Path to the MNAAS Java property file (`$Mnaas_java_propertypath`). <br>3. Log file destination (`$GenevaLoaderJavaLogPath`). <br>4. Flag `N` (likely “no‑email” or “no‑dry‑run”). |
| `#java -jar $GBSLoaderJarPath …` | Commented out | Legacy command for a previous GBS loader JAR; retained for reference. |

*No functions or classes are defined in the script; it is a procedural shell wrapper.*

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables defined in `MNAAS_Java_Batch.properties` (paths, log locations). <br>• Hard‑coded billing period `06-2020`. |
| **Outputs** | • Log file written to `$GenevaLoaderJavaLogPath` (standard output/error of the Java process). <br>• Side‑effect data load performed by the Java loader (records inserted/updated in MNAAS staging tables). |
| **External Services / Resources** | • Java Runtime (JRE) on the host. <br>• The `GevevaLoader` JAR (`$GevevaLoaderJarPath`). <br>• MNAAS configuration/property files referenced via `$Mnaas_java_propertypath`. <br>• Target database(s) accessed by the loader (likely Hive/Impala or a relational DB). |
| **Assumptions** | • The property file exists and is readable. <br>• All path variables resolve to valid files/directories. <br>• The Java loader is compatible with the hard‑coded period format. <br>• No explicit error handling; the script assumes the Java process exits cleanly. |

---

### 4. Integration Points (How This Script Connects to the Rest of the System)

| Connected Component | Relationship |
|---------------------|--------------|
| **`MNAAS_Java_Batch.properties`** | Central configuration source for all batch scripts (e.g., `mnaas_tbl_load.sh`, `move_table_compute_stats.sh`). Provides consistent JAR locations, log directories, and property file paths. |
| **`GevevaLoader` JAR** | The actual ETL engine that extracts GBS data, transforms it, and loads it into MNAAS tables. Other scripts (e.g., `mnaas_tbl_load_generic.sh`) may depend on the tables populated by this loader. |
| **Downstream Mediation Scripts** | After `runGBS.sh` completes, scripts such as `move_table_compute_stats.sh` or `product_status_report.sh` are typically scheduled to run, consuming the newly loaded data for aggregation, statistics, and reporting. |
| **Scheduling / Orchestration Layer** | Usually invoked by a cron job or an enterprise scheduler (e.g., Oozie, Airflow) as part of the nightly batch window. The same scheduler also triggers the other `*_load*` and `*_report*` scripts, ensuring ordering. |
| **Legacy `GBSLoader` command (commented)** | Indicates a previous version of the loader; may still be referenced by older pipelines or documentation. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or stale property file** | Loader fails to start; silent exit if Java cannot locate required files. | Add a pre‑run check: `[[ -r $PROP_FILE ]] || { echo "Missing properties"; exit 1; }`. |
| **Hard‑coded billing period** | Requires manual script edit each month; risk of processing wrong period. | Parameterize the period (`$1`) and default to the previous month if not supplied. |
| **No exit‑code handling** | Errors in the Java loader may go unnoticed, causing downstream jobs to run on incomplete data. | Capture `$?` after the Java command; abort with non‑zero status and send an alert (e.g., email or monitoring webhook). |
| **Log file growth** | Unlimited log accumulation can fill disk. | Rotate logs via `logrotate` or include a timestamped log filename. |
| **Jar path changes** | Deployment of a new loader version may break the script if the variable is not updated. | Store jar version in the property file and validate existence before execution. |

---

### 6. Typical Execution & Debugging Steps

1. **Run the script** (as the `MNAAS` user or via the scheduler):  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/runGBS.sh
   ```

2. **Verify environment** (optional, for debugging):  
   ```bash
   set -x   # enable shell tracing
   . /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties
   env | grep -E 'GevevaLoader|Mnaas|Geneva'
   ```

3. **Check the log** after execution:  
   ```bash
   tail -n 50 "$GenevaLoaderJavaLogPath"
   ```

4. **Validate data load** (example using Hive):  
   ```bash
   hive -e "SELECT COUNT(*) FROM mnaas.gbs_staging WHERE period='2020-06';"
   ```

5. **Troubleshoot**:  
   - If the Java process exits with a non‑zero code, inspect the log for stack traces.  
   - Ensure the JAR file is executable (`chmod +x $GevevaLoaderJarPath`).  
   - Confirm database connectivity (credentials are usually embedded in the property file referenced by `$Mnaas_java_propertypath`).  

---

### 7. External Configurations & Variables

| Variable (populated by `MNAAS_Java_Batch.properties`) | Expected Value | Usage |
|------------------------------------------------------|----------------|-------|
| `GevevaLoaderJarPath` | Full path to `GevevaLoader.jar` | Executed via `java -jar`. |
| `Mnaas_java_propertypath` | Path to a secondary MNAAS Java properties file (e.g., `MNAAS_ShellScript.properties`). | Passed as argument 2 to the loader. |
| `GenevaLoaderJavaLogPath` | Directory/file for loader logs. | Argument 3; where the loader writes its log. |
| (Potentially) `GBSLoaderJarPath`, `GBSLoaderJavaLogPath` | Legacy variables (currently unused). | Present in commented command; may be needed if the older loader is re‑enabled. |

*If any of these variables are missing, the script will fail at runtime.*

---

### 8. Suggested Improvements (TODO)

1. **Parameterize the billing period** – accept a command‑line argument (e.g., `runGBS.sh 2020-06`) and default to the previous month when omitted.  
2. **Add robust error handling** – capture the Java process exit code, log a concise status message, and exit with a non‑zero code to signal failure to the scheduler/monitoring system.  

---