**# Summary**  
HiveQL script that extracts the latest SIM‑inventory status for each VIN/ICCID pair, enriches it with KYC mapping data, and overwrites a target Hive table (parameterized via `${hiveconf:hivetable}`). Used nightly in the Move‑Mediation pipeline to provide up‑to‑date KYC‑RNR status for downstream analytics and balance‑update jobs.

**# Key Components**  
- `sim_input` CTE – selects distinct SIM, product status, plan name, and plan start date from `move_sim_inventory_status` for the most recent partition.  
- `msisdn_kyc_mapping` CTE – joins `sim_input` with `kyc_mapping_snapshot` on ICCID, adds KYC fields, and computes `row_number()` partitioned by VIN & ICCID ordered by `lastupdatedate` (most recent record first).  
- `INSERT OVERWRITE` – writes the deduplicated rows (`rownum = 1`) to the target Hive table `${hiveconf:hivetable}` with a fixed column order.

**# Data Flow**  
| Stage | Source | Transformation | Destination |
|-------|--------|----------------|-------------|
| 1 | `mnaas.move_sim_inventory_status` (Oracle → Hive external table) | Filter `prod_status` ∈ {Active, Suspended, Terminated}; keep only latest `partition_date`; select distinct SIM fields | `sim_input` CTE |
| 2 | `mnaas.kyc_mapping_snapshot` (Hive) | Inner join on `iccid = sim`; project KYC columns; compute `row_number()` per VIN/ICCID | `msisdn_kyc_mapping` CTE |
| 3 | `msisdn_kyc_mapping` | Filter `rownum = 1` (latest per VIN/ICCID) | Overwrites `${hiveconf:hivetable}` (Hive/Impala table) |

**Side Effects** – Table overwrite replaces prior nightly snapshot; no external queues or services invoked.

**# Integrations**  
- Consumes Hive external tables populated by Sqoop imports of Oracle tables (`move_sim_inventory_status`, `kyc_mapping_snapshot`).  
- Produces a Hive/Impala table consumed by downstream Move‑Mediation jobs (e.g., balance‑update, KYC‑RNR reporting).  
- Parameter `${hiveconf:hivetable}` is set by the calling orchestration layer (Oozie/Airflow/cron) to route output to the appropriate environment (dev/prod).

**# Operational Risks**  
- **Stale partition selection** – If `partition_date` is not updated, script may process outdated inventory. *Mitigation*: Validate that `max(partition_date)` is within expected freshness window before execution.  
- **Data skew on join** – Large VIN/ICCID cardinality may cause long shuffle. *Mitigation*: Ensure tables are bucketed/partitioned on `iccid` and `vin`.  
- **Overwrite loss** – Accidental mis‑configuration of `${hiveconf:hivetable}` could overwrite the wrong table. *Mitigation*: Enforce naming conventions and add a pre‑run sanity check (e.g., `DESCRIBE FORMATTED`).  

**# Usage**  
```bash
# Example invocation via Hive CLI
hive -hiveconf hivetable=move_kyc_sim_inventory_rnr_snapshot \
     -f move_kyc_sim_inventory_rnr_data.hql

# Debug: run with EXPLAIN to view execution plan
hive -e "EXPLAIN SELECT ... FROM ..."   # replace with script body
```
For step‑through debugging, set `set hive.exec.dynamic.partition=true;` and `set hive.exec.dynamic.partition.mode=nonstrict;` to allow partition creation if the target table is partitioned.

**# Configuration**  
- Hive configuration variable `hivetable` (passed via `-hiveconf`).  
- Implicit dependencies on Hive metastore definitions for `mnaas.move_sim_inventory_status` and `mnaas.kyc_mapping_snapshot`.  
- No external config files; relies on default Hive execution settings (e.g., `hive.exec.reducers.max`).

**# Improvements**  
1. **Add freshness guard** – Insert a pre‑run check that `max(partition_date)` is within the last 24 hours; abort if stale.  
2. **Materialize intermediate CTEs** – Persist `sim_input` and `msisdn_kyc_mapping` as temporary tables to enable query profiling and reuse in downstream scripts.