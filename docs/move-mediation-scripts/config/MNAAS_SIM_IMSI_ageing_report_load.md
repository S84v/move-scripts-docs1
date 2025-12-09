# Summary
`MNAAS_SIM_IMSI_ageing_report_load.properties` is a Bash‑sourced configuration fragment that defines runtime constants for the “SIM‑IMSI ageing report load” step of the MNAAS daily‑processing pipeline. It imports common properties, optionally enables Bash debugging, and supplies process‑status file path, log file name, script name, target Hive tables, and the three Hive INSERT queries executed by `MNAAS_SIM_IMSI_ageing_report_load.sh`.

# Key Components
- **Source statement** – loads global environment variables from `MNAAS_CommonProperties.properties`.
- **`setparameter`** – optional Bash `set -x` flag for debugging.
- **Process‑status variable** – `MNAAS_SIM_IMSI_ageing_report_load_ProcessStatusFileName`.
- **Log path variable** – `MNAAS_SIM_IMSI_ageing_report_load_logpath` (date‑suffixed).
- **Script name variable** – `MNAAS_SIM_IMSI_ageing_report_load_Script`.
- **Target Hive tables** – `SIM_IMSI_ageing_traffic_table`, `SIM_IMSI_ageing_sim_table`, `SIM_IMSI_report_table`.
- **Hive query strings** – `traffic_table_load_query`, `sims_table_load_query`, `report_table_load_query`.

# Data Flow
| Stage | Input | Transformation | Output |
|-------|-------|----------------|--------|
| 1 | Raw daily traffic data (`mnaas.traffic_details_raw_daily`) and org details (`mnaas.org_details`) | CTE `traffic` selects latest call per SIM, extracts IMSI prefix, inserts into `sim_imsi_ageing_traffic` | Hive table `sim_imsi_ageing_traffic` |
| 2 | Active SIMs (`mnaas.actives_raw_daily`) and latest inventory status (`mnaas.move_sim_inventory_status`) | CTE `sims` + `inv` join, selects latest SIM record, inserts into `sim_imsi_ageing_sims` | Hive table `sim_imsi_ageing_sims` |
| 3 | Pre‑built view `mnaas.v_sim_imsi_lifecycle_ageing_imsi_usage_report` | Direct INSERT into `sim_imsi_lifecycle_ageing_imsi_usage_report` | Hive report table `sim_imsi_lifecycle_ageing_imsi_usage_report` |

Side effects: updates three Hive tables; writes process‑status file; appends to dated log file.

External services: Hadoop/Hive cluster, HDFS for log/status files, underlying metastore.

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides `$MNAASConfPath`, `$MNAASLocalLogPath`, Hive connection settings.
- **`MNAAS_SIM_IMSI_ageing_report_load.sh`** – consumes all variables defined here; executes the three Hive queries via `hive -e`.
- Downstream consumers: any reporting or analytics jobs that read the three target Hive tables.

# Operational Risks
- **Query failure** – malformed Hive SQL or missing source tables → job aborts; mitigate with schema validation and test queries in dev.
- **Date partition mismatch** – traffic query filters on `partition_date like '2024-%'`; hard‑coded year may cause data gaps; mitigate by parameterizing year.
- **Debug flag leakage** – leaving `setparameter='set -x'` enabled in production may expose sensitive values in logs; mitigate by ensuring it is commented out in prod.
- **Concurrent execution** – multiple instances could write to same status/log files; mitigate with lock files or unique run identifiers.

# Usage
```bash
# Load configuration
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/config/MNAAS_SIM_IMSI_ageing_report_load.properties

# Optional: enable Bash debugging
# set -x   # uncomment setparameter line if needed

# Execute the load script
bash $MNAAS_SIM_IMSI_ageing_report_load_Script
```
To debug, uncomment the `setparameter='set -x'` line, then re‑source the file before running the script.

# Configuration
- **Referenced env files**: `MNAAS_CommonProperties.properties` (global paths, Hive connection strings).
- **Variables defined**:  
  - `$MNAASConfPath` – base config directory (from common properties).  
  - `$MNAASLocalLogPath` – local log directory (from common properties).  
  - `$MNAAS_SIM_IMSI_ageing_report_load_ProcessStatusFileName` – status file location.  
  - `$MNAAS_SIM_IMSI_ageing_report_load_logpath` – log file with date suffix.  
  - `$MNAAS_SIM_IMSI_ageing_report_load_Script` – script name to invoke.

# Improvements
1. **Parameterize date filter** – replace hard‑coded `'2024-%'` in `traffic_table_load_query` with a variable (e.g., `${PROCESS_DATE}`) to support rolling daily runs.
2. **Add validation step** – implement a pre‑execution check that verifies existence of source tables and required HDFS directories; abort with clear error if missing.