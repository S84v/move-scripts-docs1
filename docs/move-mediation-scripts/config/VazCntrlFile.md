# Summary
`VazCntrlFile.properties` is a Bash‑style configuration artifact used by the Move mediation pipeline to store the “last processed date” for aggregation tables. The timestamp is read by downstream jobs to determine the start point for incremental processing and is written back after successful runs, ensuring idempotent data loads.

# Key Components
- **`creatdt`** – ISO‑8601 timestamp (with millisecond precision) representing the last successful aggregation run.  
- **`TED`** – Human‑readable comment mirroring the purpose of `creatdt`.  
- Inline comment block – audit trail indicating when the file was last updated.

# Data Flow
| Phase | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| **Read** | `VazCntrlFile.properties` | Jobs source the file (`. /path/VazCntrlFile.properties`) to obtain `creatdt`. | In‑memory variable `creatdt`. | None |
| **Write** | Updated timestamp (current job run) | Job overwrites the file with a new `creatdt` value and updates the comment header. | Modified `VazCntrlFile.properties`. | File I/O; potential race if concurrent jobs write. |
| **Audit** | Static comment lines | Remain unchanged except for the “last updated” timestamp comment. | Updated comment header. | Provides traceability. |

External services: none directly; the timestamp drives Hive/Sqoop jobs that read/write aggregation tables.

# Integrations
- **`Sim_Inventory_Summary.sh`**, **`Tableau_security_Sqoop.sh`**, **`usage_overage_charge.sh`**, and any other aggregation‑oriented driver scripts that source `MNAAS_CommonProperties.properties` subsequently source this file to obtain `creatdt`.  
- **Hive**: `creatdt` is used in `WHERE` clauses to limit incremental loads.  
- **Sqoop**: May be passed as a parameter to control export windows.  

# Operational Risks
- **Race Condition**: Simultaneous jobs may overwrite `creatdt`, causing lost updates. *Mitigation*: Serialize access via a lock file or use a central metadata store (e.g., Hive metastore).  
- **Format Drift**: Manual edits could break the expected `yyyy-MM-dd HH:mm:ss.S` pattern. *Mitigation*: Validate format at job start; enforce via schema‑checking script.  
- **File Corruption**: Disk I/O failure could render the file unreadable. *Mitigation*: Keep a backup copy and implement retry logic.

# Usage
```bash
# Source the control file in a driver script
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/VazCntrlFile.properties

# Access the timestamp
echo "Last processed: $creatdt"

# After successful aggregation, update the file
new_ts=$(date +"%Y-%m-%d %H:%M:%S.0")
cat > /app/hadoop_users/MNAAS/MNAAS_Property_Files/VazCntrlFile.properties <<EOF
#UPDATED THE LAST PROCESSED DATE FOR AGGREGATION TABLES
#$(date -u +"%a %b %d %H:%M:%S GMT %Y")
creatdt=$new_ts
TED=THE LAST PROCESSED DATE FOR AGGREGATION TABLES
EOF
```

# Configuration
- **Referenced Config**: `MNAAS_CommonProperties.properties` (global defaults).  
- **Environment Variables**: None defined within this file; it only defines key/value pairs.  
- **File Path**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/VazCntrlFile.properties`.

# Improvements
1. **Atomic Update Mechanism** – Replace in‑place overwrite with `mv` of a temp file to guarantee atomicity.  
2. **Centralized Metadata Store** – Migrate `creatdt` to a Hive metastore table or Zookeeper node to eliminate file‑based race conditions and enable richer queries.