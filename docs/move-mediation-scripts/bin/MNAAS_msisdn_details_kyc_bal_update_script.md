# Summary
The script extracts the latest active SIM inventory, enriches API balance‑update records with KYC and organizational data, and overwrites a Hive table (name supplied via the `hivetable` HiveConf variable) partitioned by `notification_event_month`. It is executed nightly in the Move‑Mediation pipeline to refresh balance‑related reporting tables.

# Key Components
- **CTE `active_sims`** – selects active SIM rows from `mnaas.move_sim_inventory_status` for the most recent `partition_date`.
- **CTE `t2`** – left‑joins `mnaas.api_balance_update` (`ai`) with `mnaas.kyc_mapping_snapshot` (`kyc`) on `iccid`.
- **INSERT OVERWRITE** – writes the final result set into `mnaas.${hiveconf:hivetable}` with dynamic partition `notification_event_month`.
- **JOIN logic** – inner join between `t2` (API + KYC) and `active_sims` on `sim = iccid`; left join with `mnaas.org_details` to resolve `customername`.
- **Column mapping** – explicit selection and aliasing of fields required for downstream balance‑event analytics.

# Data Flow
| Stage | Source | Transformation | Destination |
|-------|--------|----------------|-------------|
| 1 | `mnaas.move_sim_inventory_status` | Filter `prod_status='Active'` and retain rows for the max `partition_date` | CTE `active_sims` |
| 2 | `mnaas.api_balance_update` + `mnaas.kyc_mapping_snapshot` | Left join on `iccid`; retain all API rows with optional KYC fields | CTE `t2` |
| 3 | `t2` + `active_sims` + `mnaas.org_details` | Inner join on `sim=iccid`; left join on org ID extracted from `tenancy_id` | Final result set |
| 4 | Final result set | `INSERT OVERWRITE` with dynamic partition `notification_event_month` (derived from first 7 chars of `event_time`) | `mnaas.${hiveconf:hivetable}` (partitioned table) |

**Side Effects** – Overwrites the target Hive table partition; no external I/O beyond Hive metastore.

**External Services** – Hive (or Spark‑SQL) execution engine; Hive metastore for table metadata.

# Integrations
- **Preceding scripts** – `med_traffic_file_recon.hql` and other nightly ingestion jobs populate `api_balance_update` and `move_sim_inventory_status`.
- **Subsequent consumers** – downstream reporting dashboards and batch jobs query `mnaas.<target_table>` for balance‑event metrics.
- **Configuration** – Relies on the HiveConf variable `hivetable` set by the orchestration layer (e.g., Airflow, Oozie) to resolve the destination table name.

# Operational Risks
- **Missing/incorrect `hivetable` variable** – leads to job failure or data written to an unintended table. *Mitigation*: validate HiveConf before execution; fail fast with clear error.
- **Stale `partition_date` logic** – if `move_sim_inventory_status` lacks a recent partition, `active_sims` may be empty, causing zero rows written. *Mitigation*: add fallback to previous partition or alert on empty result.
- **Data skew on `sim=iccid` join** – can cause long runtimes. *Mitigation*: enable map‑side join or broadcast small tables; monitor job duration.
- **Schema drift** – changes in source tables (new columns, renamed fields) break column list. *Mitigation*: maintain schema versioning and unit tests.

# Usage
```bash
# Example execution via Hive CLI
hive \
  -hiveconf hiveTable=med_cust_balance_monthly \
  -f move-mediation-scripts/bin/MNAAS_msisdn_details_kyc_bal_update_script.hql

# Debug: run with explain and limited date range
hive -e "EXPLAIN SELECT * FROM mnaas.move_sim_inventory_status LIMIT 10;" 
```
*Replace `med_cust_balance_monthly` with the target table name defined in the production catalog.*

# Configuration
- **HiveConf variable** `hivetable` – name of the destination Hive table (must exist with partition `notification_event_month`).
- **Hive session settings** (typically set upstream):  
  - `hive.exec.dynamic.partition=true`  
  - `hive.exec.dynamic.partition.mode=nonstrict`  
  - `hive.exec.max.dynamic.partitions.pernode` (adjusted for load)
- No external config files referenced directly in the script.

# Improvements
1. **Parameter Validation** – Add a pre‑execution block that checks existence of `${hiveconf:hivetable}` and aborts with a descriptive message if missing.
2. **Dynamic Partition Pruning** – Replace `substring(trim(api.event_time),1,07)` with `date_format(to_timestamp(api.event_time), 'yyyy-MM')` and enable Hive’s partition pruning to reduce scan overhead.