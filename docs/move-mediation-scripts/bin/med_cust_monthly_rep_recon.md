# Summary
The `med_cust_monthly_rep_recon.hql` script prepares a Hive‑SQL job that aggregates monthly customer‑level mediation data for a given month (`filemonth`). It selects the month parameter, then overwrites the partitioned table `mnaas.med_cust_monthly_rep_recon` with a row per `secs_id`, populating usage and CDR counts for data, SMS, and voice, and annotates each row with a “Geneva_Automation” flag based on the SECS ID and product‑code presence.

# Key Components
- **Hive settings** – forces Spark execution engine and enables dynamic partitioning (`nonstrict` mode).  
- **`select ${hiveconf:filemonth};`** – echoes the supplied month for logging/debugging.  
- **`INSERT OVERWRITE … PARTITION (partition_month)`** – writes the result set into the target table, creating/replacing the partition identified by `partition_month`.  
- **Source table** – `mnaas.new_ra_cust_monthly_rep` (aliased `cust`).  
- **Lookup sub‑query** – `mnaas.gen_usage_product` (aliased `prd`) to determine if a SECS ID has an associated product code.  
- **Derived columns** –  
  - `Geneva_Automation` (categorical flag).  
  - `partition_month` (derived as `concat(month,"-01")`).  
- **Null handling** – `nvl` used to replace nulls with defaults (`'NA'` for org name, `0` for numeric metrics).

# Data Flow
| Step | Input | Transformation | Output |
|------|-------|----------------|--------|
| 1 | Hive configuration variable `filemonth` (e.g., `2023-07`) | Passed to query via `${hiveconf:filemonth}` | Used in `WHERE month = ${hiveconf:filemonth}` and partition key |
| 2 | `mnaas.new_ra_cust_monthly_rep` rows for the target month | Selects columns, applies `nvl`, joins with product lookup, computes flags | Row set with 22 columns (excluding the first column which is the month) |
| 3 | `mnaas.gen_usage_product` (distinct `secs_code` where `plan_termination_date` is null) | Left outer join on `cust.secs_id = prd.secs_code` | Determines presence/absence of product code |
| 4 | Result set | `INSERT OVERWRITE` into `mnaas.med_cust_monthly_rep_recon` partition `partition_month` | Persisted CSV‑compatible Hive table partition |

Side effects: overwrites existing partition data; may trigger downstream downstream jobs that read this table.

External services: Hive on Spark execution engine; underlying Hadoop/HDFS storage for the target table.

# Integrations
- **Java entry point** `ReportWriter.exportReport` loads this HQL file via `TextReader`, substitutes `${hiveconf:filemonth}` with the `reportDate` argument, and executes it through a Hive JDBC connection.  
- **Reporting pipeline** – the generated partition is later consumed by downstream analytics or reconciliation jobs that expect the `med_cust_monthly_rep_recon` table to be refreshed each month.  
- **Configuration** – the script relies on Hive configuration properties set by the calling Java process (e.g., `hiveconf:filemonth`).

# Operational Risks
- **Partition overwrite** – accidental re‑run with wrong `filemonth` will delete correct data. *Mitigation*: enforce idempotent run checks, retain backup snapshots.  
- **Hard‑coded SECS IDs** (`41648,37226,41218`) for “Not Billable - MVNO”. Future changes require script edit and redeployment. *Mitigation*: externalize list to a config table or property file.  
- **Null handling assumptions** – `nvl` defaults may mask data quality issues. *Mitigation*: add data‑quality validation steps before overwrite.  
- **Dynamic partition mode** – `nonstrict` permits partitions without explicit values; mis‑typed `partition_month` could create malformed partitions. *Mitigation*: validate `partition_month` format (`YYYY-MM-01`) before insert.

# Usage
```bash
# Set Hive variable for the month (format YYYY-MM)
hive -hiveconf filemonth=2023-07 -f med_cust_monthly_rep_recon.hql
```
Or via Java:
```java
ReportWriter writer = new ReportWriter();
writer.exportReport("MED_CUST_MONTHLY_RECON", "2023-07", "12345");
```
Debugging tip: the initial `select ${hiveconf:filemonth};` prints the month in the Hive log.

# Configuration
- **Hive variable** `filemonth` – month to process (e.g., `2023-07`).  
- **Hive execution engine** – forced to `spark` within the script; requires Spark‑enabled Hive cluster.  
- **Dynamic partition settings** – set in script; can be overridden by session defaults if needed.  
- **External lookup table** `mnaas.gen_usage_product` – must be up‑to‑date for correct `Geneva_Automation` flag.

# Improvements
1. **Externalize static SECS ID list** – move `41648,37226,41218` to a reference table or property file to avoid code changes for business rule updates.  
2. **Add pre‑insert validation** – implement a Hive `SELECT COUNT(*)` check on the source month and on the generated `partition_month` format; abort if mismatches are detected, and log a warning.