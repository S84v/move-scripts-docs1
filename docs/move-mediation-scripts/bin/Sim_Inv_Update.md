**Sim_Inv_Update.sh – High‑Level Documentation**

---

### 1. Summary
`Sim_Inv_Update.sh` is a thin wrapper that launches the Java application `SIMInvUpdate.jar` with a fixed heap size (8 GB) and two numeric arguments (`500000` and `1300000`). In production the jar processes SIM inventory data – typically reading staged files or database tables, applying business rules, and persisting the updated inventory to the target data store. The script is part of the “SIM inventory” pipeline and is invoked after the staging‑area load (e.g., by `MNAAS_non_move_files_from_staging.sh` or `Move_Sim_Inventory_Aggr.sh`) to bring the canonical inventory up‑to‑date.

---

### 2. Key Components & Responsibilities
| Component | Responsibility |
|-----------|----------------|
| **Sim_Inv_Update.sh** (shell script) | Sets Java memory limits, invokes the JAR with required parameters, and propagates the exit status. |
| **SIMInvUpdate.jar** (Java application) | *Main class* (unknown name) reads input data, performs transformation/validation of SIM inventory records, writes results to the production inventory tables (or HDFS), and logs processing statistics. |
| **Java runtime** (`java -Xms8192m -Xmx8192m`) | Provides the JVM with a minimum and maximum heap of 8 GB, ensuring sufficient memory for large batch processing. |
| **Arguments `500000` `1300000`** | Likely represent batch size limits (e.g., max rows per transaction and max rows per job). Exact meaning is defined inside the JAR. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions
| Category | Details |
|----------|---------|
| **Inputs** | • Implicit input sources read by the JAR (e.g., staging tables in Hive/Impala, HDFS files under `/data/staging/sim_inventory/`). <br>• The two numeric arguments (batch limits). |
| **Outputs** | • Updated SIM inventory tables (e.g., `dim_sim_inventory`, `fact_sim_usage`). <br>• Log files written to the standard output or a configured log directory (often `/var/log/mnaas/`). |
| **Side Effects** | • May trigger downstream processes that depend on fresh inventory (e.g., billing, reporting). <br>• Consumes significant JVM heap; may affect other jobs on the same node if resources are over‑committed. |
| **Assumptions** | • Java 8+ is installed and `JAVA_HOME` points to a compatible JRE. <br>• The executing user (`hadoop_users/MNAAS`) has read/write access to the JAR and any HDFS/Hive resources. <br>• Environment variables for Hadoop (`HADOOP_CONF_DIR`, `HIVE_CONF_DIR`) are correctly set by the login shell or a wrapper script. |

---

### 4. Integration with Other Scripts / Components
| Connected Script | Interaction |
|------------------|-------------|
| `MNAAS_non_move_files_from_staging.sh` | Loads raw SIM files into staging; `Sim_Inv_Update.sh` runs **after** this step to transform staged data. |
| `Move_Sim_Inventory_Aggr.sh` | Performs aggregation on the inventory after the update; typically scheduled **after** `Sim_Inv_Update.sh` completes successfully. |
| `MNAAS_report_data_loading.sh` | May include a status check for the SIM inventory update; consumes exit code/logs from this script. |
| `MNAAS_sqoop_bareport.sh` / `Org_details_Sqoop.sh` | Provide auxiliary data (e.g., customer‑org mapping) that the JAR may join against. |
| Monitoring / Scheduler (e.g., Oozie, Airflow) | Calls `Sim_Inv_Update.sh` as a task node; expects a zero exit code for success. |

---

### 5. Operational Risks & Recommended Mitigations
| Risk | Mitigation |
|------|------------|
| **Out‑of‑memory JVM crash** (heap too small for data volume) | – Periodically review batch arguments; increase `-Xmx` if node memory permits. <br>– Add a pre‑run check of available system memory (`free -m`). |
| **Silent failure of the JAR** (non‑zero exit not captured) | – Ensure the script propagates `$?` (`exit $?.` already implicit). <br>– Wrap the java call in `set -e` or explicit error handling to abort the workflow on failure. |
| **Missing or corrupted JAR** | – Validate checksum of `SIMInvUpdate.jar` during deployment. <br>– Add a file‑existence test before execution (`[ -f … ] || exit 1`). |
| **Incorrect arguments** (batch limits changed upstream) | – Document the meaning of the two numbers; consider moving them to a config file or environment variable. |
| **Permission issues on HDFS/Hive** | – Run the script as the dedicated Hadoop user; verify Kerberos tickets (`kinit`) are valid before launch. |
| **Log flooding** (large stdout) | – Redirect stdout/stderr to a rotating log file (`>> /var/log/mnaas/sim_inv_update.log 2>&1`). |

---

### 6. Running & Debugging Guide
1. **Standard execution**  
   ```bash
   $ cd /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars
   $ /path/to/move-mediation-scripts/bin/Sim_Inv_Update.sh
   ```
   The script returns the exit status of the Java process (`0` = success).

2. **Verbose/debug mode** (add JVM debug flags temporarily)  
   ```bash
   java -Xms8192m -Xmx8192m -verbose:gc -jar SIMInvUpdate.jar 500000 1300000
   ```
   Observe GC logs and any stack traces printed to the console.

3. **Check logs**  
   - If the JAR writes to a log file, tail it: `tail -f /var/log/mnaas/sim_inv_update.log`.  
   - Verify Hive/HDFS tables were updated: `hive -e "SELECT COUNT(*) FROM dim_sim_inventory;"`.

4. **Validate exit code**  
   ```bash
   $ ./Sim_Inv_Update.sh
   $ echo $?   # should be 0
   ```

5. **Common troubleshooting steps**  
   - Confirm Java version: `java -version`.  
   - Verify JAR integrity: `sha256sum SIMInvUpdate.jar`.  
   - Ensure required Hadoop configs are on the classpath (check `$HADOOP_CONF_DIR`).  

---

### 7. External Configuration / Environment Variables
| Variable / File | Usage |
|-----------------|-------|
| `JAVA_HOME` | Determines which `java` binary is invoked. |
| `HADOOP_CONF_DIR`, `HIVE_CONF_DIR` | Provide Hadoop/Hive connection details to the JAR (via the JVM’s classpath). |
| `PATH` | Must include the directory containing the `java` executable. |
| **Hard‑coded paths** (`/app/hadoop_users/MNAAS/.../SIMInvUpdate.jar`) | The script assumes a static deployment layout; any change requires script modification. |
| **Batch arguments** (`500000`, `1300000`) | Currently hard‑coded; could be externalized to a properties file (e.g., `/etc/mnaas/sim_inv_update.conf`). |

---

### 8. Suggested Improvements (TODO)
1. **Parameterize arguments and paths** – Move the two numeric arguments and the JAR location into a configurable properties file or environment variables to avoid code changes for batch‑size tuning or redeployment.  
2. **Add robust error handling** – Wrap the `java` call with `set -euo pipefail` and explicit checks (`if [ $? -ne 0 ]; then echo "SIMInvUpdate failed"; exit 1; fi`) to guarantee the scheduler detects failures promptly.  

---