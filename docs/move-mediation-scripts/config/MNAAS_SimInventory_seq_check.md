# Summary
Defines Bash‑sourced configuration for the SIM‑Inventory sequence‑number validation step in the MNAAS daily‑processing pipeline. It imports shared common properties and declares the HDFS directory identifier used by downstream validation scripts.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  Loads global environment variables, log locations, and Hadoop/Hive connection settings.
- **Variable definition** – `Dname_MNAAS_SimInventory_seqno_check`  
  Holds the logical HDFS directory name for the SIM‑Inventory sequence‑number check files.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| Config load | `MNAAS_CommonProperties.properties` | Bash `source` merges global vars into current shell | Environment variables + `Dname_MNAAS_SimInventory_seqno_check` | None |
| Validation script (e.g., `MNAAS_SimInventory_seq_check.sh`) | HDFS path derived from `Dname_MNAAS_SimInventory_seqno_check` | Reads current and historical sequence files, compares against expected range | Status file, log entries, missing‑seq report in HDFS | HDFS read/write, log rotation |

# Integrations
- **MNAAS_CommonProperties.properties** – central repository for all pipeline constants (HDFS base paths, log directories, Hive/Impala credentials).
- **MNAAS_SimInventory_seq_check.sh** – consumes `Dname_MNAAS_SimInventory_seqno_check` to locate sequence files.
- Downstream aggregation jobs that reference the same HDFS directory for consistency checks.

# Operational Risks
- **Typo in variable assignment** – duplicate `=` may cause the variable to be empty or incorrectly set, leading to path resolution failures.  
  *Mitigation*: Validate variable after sourcing; add unit test for config parsing.
- **Missing common properties file** – source failure aborts script execution.  
  *Mitigation*: Guard with existence check and fallback defaults.
- **Hard‑coded absolute path** – reduces portability across environments.  
  *Mitigation*: Parameterize base directory via environment variable.

# Usage
```bash
# Load configuration in a Bash session
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SimInventory_seq_check.properties

# Verify variable resolution
echo "$Dname_MNAAS_SimInventory_seqno_check"

# Run the validation script (example)
bash MNAAS_SimInventory_seq_check.sh
```

# Configuration
- **Environment Variables** (populated by common properties):  
  `HDFS_ROOT`, `LOG_DIR`, `PROCESS_STATUS_DIR`, `HIVE_JDBC_URL`, etc.
- **Referenced Config Files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - Other step‑specific property files (e.g., `MNAAS_SimInventory_tbl_Load.properties`).

# Improvements
1. **Correct variable assignment** – replace erroneous line with:  
   `Dname_MNAAS_SimInventory_seqno_check=MNAAS_SimInventory_seqno_check`
2. **Add validation block** to ensure the sourced file exists and the variable is non‑empty, exiting with a clear error code if checks fail.