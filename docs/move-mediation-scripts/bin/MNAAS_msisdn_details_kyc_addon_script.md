# Summary
The script creates (if absent) and populates a Hive table containing detailed MSISDN‑addon information enriched with KYC data. It extracts active SIM inventory, joins it to KYC snapshots, and merges the result with API‑notification addon records. The final dataset is partitioned by addon expiration date and stored in the `mnaas` database for downstream reporting and analytics.

# Key Components
- **Hive session settings** – enable dynamic partitioning and increase partition limits.  
- **`create table if not exists ${hiveconf:hivetable}`** – defines target table schema and partition column `partition_date`.  
- **CTE `active_sims`** – selects active SIMs from `mnaas.move_sim_inventory_status` within a configurable date window.  
- **CTE `t2`** – left‑joins `active_sims` with `kyc_mapping_snapshot` to attach VIN, make, model, device type, and country of sale.  
- **`insert into mnaas.${hivetable} partition(partition_date)`** – inserts enriched addon records from `api_notification_addon`, joining to `t2` and `mnaas.org_details` for organization name.  
- **Hive variables** – `${hiveconf:hivetable}`, `${hiveconf:start_date}`, `${hiveconf:ending_date}` used for table name and date range.

# Data Flow
| Stage | Input | Transformation | Output |
|-------|-------|----------------|--------|
| 1 | `mnaas.move_sim_inventory_status` (filtered by `prod_status='Active'` and `partition_date` window) | Filter → CTE `active_sims` | Active SIM rows |
| 2 | `active_sims` + `kyc_mapping_snapshot` | Left join on `sim = iccid` | Enriched SIM rows (`t2`) |
| 3 | `api_notification_addon` + `t2` + `mnaas.org_details` | Inner join on `iccid`, left join on org ID, compute `partition_date` from `expiration_date` | Final record set |
| 4 | Final record set | Insert overwrite (dynamic partition) into `${hiveconf:hivetable}` | Partitioned Hive table `mnaas.<hivetable>` |

Side effects: overwrites partitions for the supplied `partition_date` values; no external queues.

# Integrations
- **Upstream**: `move_sim_inventory_status` and `kyc_mapping_snapshot` tables populated by SIM inventory and KYC ingestion pipelines.  
- **Downstream**: Tables created by this script are consumed by reporting jobs, BI dashboards, and ad‑hoc analytics that require MSISDN‑addon and KYC correlation.  
- **Configuration**: Relies on Hive variables passed via `hiveconf` (e.g., from orchestration tools like Oozie, Airflow, or custom shell wrappers).

# Operational Risks
- **Partition Overwrite Collisions** – concurrent runs with overlapping `partition_date` may cause data loss. *Mitigation*: enforce single‑instance execution per date range or use `INSERT INTO` with deduplication.  
- **Stale KYC Data** – if `kyc_mapping_snapshot` is not refreshed before execution, records may contain outdated VIN/info. *Mitigation*: schedule KYC refresh ahead of this job.  
- **Dynamic Partition Limits** – exceeding `hive.exec.max.dynamic.partitions` can abort the job. *Mitigation*: monitor partition count; adjust limits if needed.  
- **Data Skew** – large volume for a single `partition_date` may cause reducer overload. *Mitigation*: add `CLUSTER BY` or `DISTRIBUTE BY` on high‑cardinality columns.

# Usage
```bash
# Example invocation via Hive CLI
hive -hiveconf hivetable=msisdn_kyc_addon \
     -hiveconf start_date=2024-01-01 \
     -hiveconf ending_date=2024-01-31 \
     -f MNAAS_msisdn_details_kyc_addon_script.hql
```
For debugging, prepend `set hive.exec.print.summary=true;` and `EXPLAIN` the final `INSERT` query.

# Configuration
- **Hive variables** (required):
  - `hivetable` – target table name (e.g., `msisdn_kyc_addon`).
  - `start_date` – start of inventory partition window (format `yyyy-MM-dd`).
  - `ending_date` – end of inventory partition window (format `yyyy-MM-dd`).
- **Session settings** (hard‑coded in script):
  - `hive.exec.dynamic.partition=true`
  - `hive.exec.dynamic.partition.mode=nonstrict`
  - `hive.exec.max.dynamic.partitions=50000`
  - `hive.exec.max.dynamic.partitions.pernode=20000`

# Improvements
1. **Idempotent Load** – replace `INSERT OVERWRITE` with `MERGE` (or `INSERT INTO` + deduplication) to avoid accidental data loss on overlapping runs.  
2. **Parameter Validation** – add a pre‑execution block that checks required Hive variables and aborts with a clear error if missing or malformed.